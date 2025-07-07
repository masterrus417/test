import io
import os
import tempfile

from django.conf import settings
from django.core.files.storage import FileSystemStorage

from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

# Получаем настройки с дефолтами
NONCE_LEN     = getattr(settings, 'FILE_ENCRYPTION_NONCE_LEN', 12)
TAG_LEN       = getattr(settings, 'FILE_ENCRYPTION_TAG_LEN', 16)
ENC_BUF_MB    = getattr(settings, 'FILE_ENCRYPTION_BUFFER_LIMIT_MB', 20)
DEC_BUF_MB    = getattr(settings, 'FILE_DECRYPTION_BUFFER_LIMIT_MB', 20)
ENC_CHUNK     = getattr(settings, 'FILE_ENCRYPTION_CHUNK_SIZE', 8192)
DEC_CHUNK     = getattr(settings, 'FILE_DECRYPTION_CHUNK_SIZE', 8192)


class EncryptingStreamWriter:
    def __init__(self, input_stream, key, buffer_limit_mb=20, chunk_size=8192, nonce_len=12):
        self.input_stream = input_stream
        self.key = key
        self.chunk_size = chunk_size
        self.nonce_len = nonce_len

        self.nonce = os.urandom(self.nonce_len)
        self.encryptor = Cipher(
            algorithms.AES(self.key),
            modes.GCM(self.nonce),
            backend=default_backend()
        ).encryptor()

        self.buffer = tempfile.SpooledTemporaryFile(
            max_size=buffer_limit_mb * 1024 * 1024
        )
        self._write_encrypted()

    def _write_encrypted(self):
        self.buffer.write(self.nonce)

        while True:
            chunk = self.input_stream.read(self.chunk_size)
            if not chunk:
                break
            self.buffer.write(self.encryptor.update(chunk))

        self.buffer.write(self.encryptor.finalize())
        self.buffer.write(self.encryptor.tag)
        self.buffer.seek(0)

    def get_stream(self):
        return self.buffer


class DecryptingStream(io.RawIOBase):
    def __init__(self, fileobj, decryptor, enc_len, chunk_size=8192):
        self._file = fileobj
        self._decryptor = decryptor
        self._remaining = enc_len
        self._chunk_size = chunk_size
        self._finalized = False

    def read(self, size=-1):
        if self._finalized:
            return b""

        if size == -1 or size is None:
            size = self._chunk_size

        size = min(size, self._chunk_size)

        if self._remaining <= 0:
            self._finalized = True
            return self._decryptor.finalize()

        to_read = min(size, self._remaining)
        chunk = self._file.read(to_read)
        self._remaining -= len(chunk)

        out = self._decryptor.update(chunk)

        if self._remaining == 0:
            out += self._decryptor.finalize()
            self._finalized = True

        return out

    def readable(self): return True
    def close(self):    self._file.close()
    def __getattr__(self, attr): return getattr(self._file, attr)


class EncryptedFileSystemStorage(FileSystemStorage):
    def _save(self, name, content):
        key = getattr(settings, 'FILE_ENCRYPTION_KEY')
        writer = EncryptingStreamWriter(
            input_stream=content,
            key=key,
            buffer_limit_mb=ENC_BUF_MB,
            chunk_size=ENC_CHUNK,
            nonce_len=NONCE_LEN
        )
        return super()._save(name, writer.get_stream())

    def _open(self, name, mode='rb'):
        encrypted = super()._open(name, mode)

        encrypted.seek(0, os.SEEK_END)
        total = encrypted.tell()
        encrypted.seek(0)

        nonce = encrypted.read(NONCE_LEN)
        enc_len = total - NONCE_LEN - TAG_LEN
        encrypted.seek(NONCE_LEN + enc_len)
        tag = encrypted.read(TAG_LEN)
        encrypted.seek(NONCE_LEN)

        key = getattr(settings, 'FILE_ENCRYPTION_KEY')
        decryptor = Cipher(
            algorithms.AES(key),
            modes.GCM(nonce, tag),
            backend=default_backend()
        ).decryptor()

        return DecryptingStream(encrypted, decryptor, enc_len, chunk_size=DEC_CHUNK)

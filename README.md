import tempfile
from django.conf import settings
from django.core.files.base import ContentFile, File
from django.core.files.storage import FileSystemStorage
from cryptography.fernet import Fernet

class EncryptedFileSystemStorage(FileSystemStorage):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fernet = Fernet(settings.FILE_ENCRYPTION_KEY)

    def _save(self, name, content):
        encrypted = self.fernet.encrypt(content.read())
        return super()._save(name, ContentFile(encrypted))

    def _open(self, name, mode='rb'):
        encrypted_file = super()._open(name, mode)
        decrypted = self.fernet.decrypt(encrypted_file.read())
        tmp = tempfile.SpooledTemporaryFile(max_size=10 * 1024 * 1024)
        tmp.write(decrypted)
        tmp.seek(0)
        return File(tmp, name)

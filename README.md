
custom_db_backend/
├── __init__.py
├── base.py                # Django DB backend wrapper
├── connection.py          # Подключение к PostgreSQL с поддержкой Kerberos
├── middleware.py          # Middleware для REMOTE_USER и keytab
├── remote_auth.py         # Кеш авторизации remote_user
└── settings_sample.py     # Пример настройки DATABASES

# __init__.py
# пустой файл

# middleware.py
import threading
import getpass
import os
from typing import Optional

remote_user_holder = threading.local()

class RemoteUserMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        remote_user = request.META.get('REMOTE_USER')
        keytab = request.META.get('KRB5_CLIENT_KTNAME')

        # fallback для локальной разработки
        if not remote_user:
            remote_user = getpass.getuser()
        if not keytab:
            keytab = os.environ.get('KRB5_CLIENT_KTNAME')

        remote_user_holder.user: Optional[str] = remote_user
        remote_user_holder.keytab: Optional[str] = keytab
        return self.get_response(request)


# remote_auth.py
import threading
import time
import requests
from typing import Dict, Tuple

_cache: Dict[str, Tuple[Dict[str, str], float]] = {}
_lock = threading.Lock()
CACHE_TTL = 600

def fetch_credentials_from_api(api_url: str, remote_user: str) -> Dict[str, str]:
    now = time.time()
    with _lock:
        entry = _cache.get(remote_user)
        if entry:
            creds, ts = entry
            if now - ts < CACHE_TTL:
                return creds

    response = requests.post(api_url, json={"remote_user": remote_user})
    response.raise_for_status()
    creds: Dict[str, str] = response.json()

    with _lock:
        _cache[remote_user] = (creds, now)

    return creds


# base.py
from django.db.backends.postgresql.base import DatabaseWrapper as PostgresWrapper
from .connection import CustomDatabaseConnection
from .remote_auth import fetch_credentials_from_api
from .middleware import remote_user_holder
import os
from typing import Any, Dict, Optional


class DatabaseWrapper(PostgresWrapper):
    def get_connection_params(self) -> Dict[str, Any]:
        settings_dict: Dict[str, Any] = self.settings_dict.copy()
        type_connection: str = settings_dict.pop("TYPE_CONNECTION", "default")
        settings_dict["connection_type"] = type_connection

        if type_connection == "default":
            return settings_dict

        if type_connection == "remote_user":
            remote_user: Optional[str] = getattr(remote_user_holder, "user", None)
            if not remote_user:
                return settings_dict

            api_url: str = settings_dict.pop("AUTH_API_URL")
            creds: Dict[str, str] = fetch_credentials_from_api(api_url, remote_user)
            settings_dict["USER"] = creds["username"]
            settings_dict["PASSWORD"] = creds["password"]
            return settings_dict

        if type_connection == "krb":
            remote_user: Optional[str] = getattr(remote_user_holder, "user", None)
            keytab: Optional[str] = getattr(remote_user_holder, "keytab", None)
            if not remote_user or not keytab:
                raise Exception("Kerberos auth requires REMOTE_USER and KRB5_CLIENT_KTNAME")

            os.environ["KRB5_CLIENT_KTNAME"] = keytab
            settings_dict["USER"] = remote_user
            settings_dict["PASSWORD"] = ""
            settings_dict["KERBEROS"] = True
            settings_dict["PORT"] = settings_dict.get("KRB_PORT", 5433)

            return settings_dict

        return settings_dict

    def get_new_connection(self, conn_params: Dict[str, Any]) -> Any:
        return CustomDatabaseConnection(conn_params)


# connection.py
import os
import subprocess
import psycopg2
from typing import Any, Dict, Optional


class CustomDatabaseConnection:
    def __init__(self, conn_params: Dict[str, Any]) -> None:
        self.connection_type: str = conn_params.pop("connection_type", "default")
        self.use_kerberos: bool = conn_params.pop("KERBEROS", False)
        self.conn: Any = self._connect(conn_params)

    def _connect(self, params: Dict[str, Any]) -> Any:
        if self.use_kerberos:
            principal: str = params.get("USER")
            self._kinit(principal)
            return psycopg2.connect(**params, options="-k")
        return psycopg2.connect(**params)

    def _kinit(self, principal: str) -> None:
        keytab: Optional[str] = os.getenv("KRB5_CLIENT_KTNAME")
        if not keytab:
            raise Exception("KRB5_CLIENT_KTNAME not set")
        result = subprocess.run(["kinit", principal, "-k", "-t", keytab], capture_output=True)
        if result.returncode != 0:
            raise Exception(f"kinit failed: {result.stderr.decode()}")

    def __getattr__(self, name: str) -> Any:
        return getattr(self.conn, name)


# settings_sample.py
DATABASES = {
    'default': {
        'ENGINE': 'custom_db_backend.base',
        'NAME': 'your_db',
        'USER': 'default_user',
        'PASSWORD': 'default_pass',
        'HOST': 'localhost',
        'PORT': '5432',
        'KRB_PORT': '5433',
        'TYPE_CONNECTION': 'remote_user',  # или 'krb', или 'default'
        'AUTH_API_URL': 'http://auth-service.local/get_creds'  # только для remote_user
    }
}

# Пример выполнения SQL-запросов с использованием ORM или сырого SQL

from django.db import connection

# Пример сырого SQL
with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM your_table WHERE id = %s", [1])
    row = cursor.fetchone()
    print(row)

# Пример ORM-запроса
# from your_app.models import YourModel
# result = YourModel.objects.raw("SELECT * FROM your_table WHERE id = %s", [1])

import io, time, zipfile, os
from django.utils import timezone
from django.utils.dateparse import parse_datetime
from django.db.models import Q
from rest_framework import status, viewsets
from rest_framework.parsers import MultiPartParser
from rest_framework.response import Response
from django.http import FileResponse
from django.core.files.base import ContentFile
from django.shortcuts import get_object_or_404
from django.conf import settings

from .models import FilePackage
from .serializers import FilePackageSerializer
from .storage import EncryptedFileSystemStorage

class FilePackageViewSet(viewsets.ModelViewSet):
    serializer_class = FilePackageSerializer
    parser_classes = [MultiPartParser]

    def get_queryset(self):
        now = timezone.now()
        return FilePackage.objects.filter(
            is_deleted=False
        ).filter(
            Q(expires_at__isnull=True) | Q(expires_at__gt=now)
        )

    def get_object(self):
        return get_object_or_404(self.get_queryset(), pk=self.kwargs['pk'])

    def create(self, request, *args, **kwargs):
        files = request.FILES.getlist('file')
        if not files:
            return Response({'error': 'No files provided'}, status=400)

        expires_raw = request.data.get('expires_at')
        expires_at = parse_datetime(expires_raw) if expires_raw else None

        storage_type = request.data.get('storage_type', 'fast')
        if storage_type not in ('fast', 'slow'):
            return Response({'error': 'Invalid storage_type'}, status=400)

        file_metadata = []
        zip_buffer = io.BytesIO()

        with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zf:
            for f in files:
                data = f.read()
                zf.writestr(f.name, data)
                file_metadata.append({
                    'filename': f.name,
                    'size': f.size,
                    'content_type': f.content_type or 'application/octet-stream'
                })
        zip_buffer.seek(0)

        location = settings.FAST_STORAGE_ROOT if storage_type == 'fast' else settings.SLOW_STORAGE_ROOT
        storage = EncryptedFileSystemStorage(location=location)

        archive_name = f"package_{int(time.time())}.zip.enc"
        storage.save(archive_name, ContentFile(zip_buffer.read()))

        obj = FilePackage.objects.create(
            archive=os.path.join('packages', archive_name),
            original_name=archive_name,
            file_list=file_metadata,
            expires_at=expires_at,
            storage_type=storage_type
        )

        return Response(self.get_serializer(obj).data, status=201)

    def destroy(self, request, *args, **kwargs):
        obj = self.get_object()
        provided_key = request.query_params.get('private_key')
        if not provided_key:
            return Response({'error': 'Missing ?private_key=...'}, status=400)
        if str(obj.private_key) != provided_key:
            return Response({'error': 'Invalid private key'}, status=403)

        obj.is_deleted = True
        obj.save()
        return Response(status=204)

    def retrieve(self, request, *args, **kwargs):
        obj = self.get_object()
        provided_key = request.query_params.get('public_key')
        if provided_key and str(obj.public_key) == provided_key:
            location = settings.FAST_STORAGE_ROOT if obj.storage_type == 'fast' else settings.SLOW_STORAGE_ROOT
            storage = EncryptedFileSystemStorage(location=location)
            fh = storage.open(obj.archive.name, 'rb')
            return FileResponse(fh, as_attachment=True,
                                filename=obj.original_name,
                                content_type='application/zip')
        return super().retrieve(request, *args, **kwargs)

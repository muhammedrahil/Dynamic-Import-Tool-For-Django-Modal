# Dynamic-Import-Tool-For-Django-Models

## Overview

The Dynamic-Import-Tool-For-Django-Models is a utility for Django that simplifies the bulk import of data from text files into Django models. It automates file reading, column mapping, error handling, and transaction management. This tool is ideal for importing large datasets into your Django application, ensuring data integrity and providing feedback through email notifications.

## Features

- **Bulk Import:** Efficiently imports data from `.txt` files into Django models.
- **Dynamic Model Mapping:** Automatically maps columns from the file to the corresponding fields in Django models.
- **Error Handling:** Captures and logs errors, allowing for robust data import with transactional integrity.
- **Transaction Management:** Uses Django transactions to ensure that each record is correctly processed and saved.
- **Email Notifications:** Notifies users of the success or failure of the import process via email.
- **File Cleanup:** Automatically deletes files after processing.

```python


import datetime
import os
import chardet
from PROJECT import settings
from app.models import BulkDump, BulkDumpFiles
import pandas as pd
from django.apps import apps
from io import StringIO
from django.db import transaction
import re
from datetime import datetime
from django.core.mail import send_mail

def bulk_import_instence(name):
    dump = BulkDump.objects.filter(name=name).order_by('-created_at').first()
    Bulkdumpfiles = BulkDumpFiles.objects.filter(bulk_dump=dump).values('file')
    return dump, Bulkdumpfiles

def print_response(prefix, msg, color="success"):
    RED = '\033[91m'
    GREEN = '\033[92m'
    END_COLOR = '\033[0m'
    if color == "success":
        print(f"{GREEN}{prefix} : {msg}{END_COLOR}")
    if color == "error":
        print(f"{RED}{prefix} : {msg}{END_COLOR}")
    return 

def dynamic_import_tool(request, instence_name):
    dump_instence, Bulkdumpfiles = bulk_import_instence(instence_name)
    models = apps.get_app_config("old_patriot_app").get_models()
    models_dict = {model._meta.db_table: model for model in models}
    Failed = False
    error = ""
    for file_obj in Bulkdumpfiles:
        file_path = file_obj.get('file')
        file_name = os.path.basename(file_path)
        file_path = os.path.join(settings.MEDIA_ROOT, file_path)
        file_name_without_extension, extension = os.path.splitext(file_name)
        if extension == '.txt':
            if file_name_without_extension in models_dict:
                model = models_dict[file_name_without_extension]
                try:
                    with open(file_path, 'r', encoding='latin-1', errors='replace') as file:
                        content = file.read()
                        content = StringIO(content)
                        df = pd.read_csv(content,sep='|')
                        column_mapping, bulk_dump, decimal, date = get_fields_db_name(model)
                        df.rename(columns=column_mapping, inplace=True)
                        df.fillna('')
                        if bulk_dump:
                            df['bulk_dump'] = dump_instence
                        cleaned_data = df.to_dict(orient="records")
                        for record in cleaned_data:
                            with transaction.atomic():
                                try:
                                    for key, value in record.items():
                                        if key in decimal:
                                            value = re.sub('[^\d.]', '', str(value))
                                            if value:
                                                record[key] = float(value)
                                        if key in date:
                                            date_format = "%m/%d/%Y %H:%M:%S"
                                            try:
                                                record[key] = datetime.strptime(value, date_format)
                                            except:
                                                record[key] = datetime.now()
                                        if value == "" or pd.isna(value) :
                                            record[key] = None
                                    model.objects.create(**record)
                                except Exception as e:
                                    print_response("error",f"{file_name_without_extension} - {str(e)}", color="error")
                        print_response("Success",file_name_without_extension)
                except Exception as e:
                    print_response("error",str(e), color="error")
                    Failed = True
                    error = str(e)
                    if os.path.exists(file_path):
                        os.remove(file_path)
        if os.path.exists(file_path):
            os.remove(file_path)
    if not Failed:
        print_response("Success",f"Old {instence_name} Import is Completed")
    BulkDumpFiles.objects.filter(bulk_dump=dump_instence).delete()
    send_mail(
        subject=f"Old {instence_name} Import is Completed" if not Failed else f"Old {instence_name} Import is Failed",
        message=f"Old {instence_name} Import is Completed" if not Failed else error,
        from_email=str(settings.DEFAULT_FROM_EMAIL),
        recipient_list=[],
        )
    return 


def get_fields_db_name(model):
    from django.db.models import DecimalField, DateTimeField
    column_mapping = {}
    decimal = []
    date = []
    bulk_dump = False
    for field in model._meta.get_fields():
        if field.name != 'bulk_dump':
            column_mapping.update({field.db_column : field.name})
            if isinstance(field, DecimalField):
                decimal.append(field.name)
            if isinstance(field, DateTimeField):
                date.append(field.name)
        else:
            bulk_dump = True
    return column_mapping, bulk_dump, decimal, date
```

### How to call dynamic_import_tool function
```python

@api_view(['POST'])
def import_tool_api(request):
    name = request.POST.get('name')
    if not name:
        return Response({"msg": "Import Name is Required"}, status=status.HTTP_400_BAD_REQUEST)
    dump = BulkDump.objects.filter(name=name).first()
    if not dump:
        dump = BulkDump.objects.create(name=name)
    files = request.FILES
    for filename, file in files.items():
        file_name = file.name
        BulkDumpFiles.objects.create(file_name=file_name,file=file,bulk_dump=dump)
    thread = threading.Thread(target=dynamic_import_tool, args=(request,name,))
    thread.start()
    return Response({"msg": "Successfully Imported"}, status=status.HTTP_200_OK)


```

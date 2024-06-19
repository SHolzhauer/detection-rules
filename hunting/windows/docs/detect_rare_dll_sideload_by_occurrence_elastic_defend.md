# Detect Rare DLL SideLoad by Occurrence - Elastic Defend

---

## Metadata

- **Author:** Elastic
- **UUID:** `bcdb7c29-1312-4974-8f2e-10ddeb09cf5c`
- **Integration:** [endpoint](https://docs.elastic.co/integrations/endpoint)
- **Language:** `ES|QL`

## Query

```sql
from logs-endpoint.events.library-*
| where @timestamp > NOW() - 7 day
| where host.os.family == "windows" and event.action == "load" and process.code_signature.status == "trusted" and dll.code_signature.status != "trusted" and dll.Ext.relative_file_creation_time <= 86400
| eval dll_folder = substring(dll.path, 1, length(dll.path) - (length(dll.name) + 1))
| eval process_folder = substring(process.executable, 1, length(process.executable) - (length(process.name) + 1))
| where process_folder is not null and dll_folder is not null and process_folder == dll_folder and process.name != dll.name
| eval dll_folder = replace(dll_folder, """([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}|ns[a-z][A-Z0-9]{3,4}\.tmp|DX[A-Z0-9]{3,4}\.tmp|7z[A-Z0-9]{3,5}\.tmp|[0-9\.\-\_]{3,})""", ""), process_folder = replace(process_folder, """([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}|ns[a-z][A-Z0-9]{3,4}\.tmp|DX[A-Z0-9]{3,4}\.tmp|7z[A-Z0-9]{3,5}\.tmp|[0-9\.\-\_]{3,})""", "")
| eval dll_folder = replace(dll_folder, """[cC]:\\[uU][sS][eE][rR][sS]\\[a-zA-Z0-9\.\-\_\$]+\\""", "C:\\\\users\\\\user\\\\"), process_folder = replace(process_folder, """[cC]:\\[uU][sS][eE][rR][sS]\\[a-zA-Z0-9\.\-\_\$]+\\""", "C:\\\\users\\\\user\\\\")
| stats host_count = count_distinct(host.id), total_count = count(*) by dll_folder, dll.name, process.name, dll.hash.sha256
/* total_count can be adjusted to higher or lower values depending on env */
| where host_count == 1 and total_count <= 10 | keep total_count, host_count, dll_folder, dll.name, process.name, dll.hash.sha256
```

## Notes

- Based on the returned results you can further investigate suspicious DLLs by sha256 and library path.
- Paths like C:\\Users\\Public and C:\\ProgramData\\ are often observed in malware employing DLL side-loading.
- Elastic Defned DLL Events include dll.Ext.relative_file_creation_time which help us limit the hunt to recently dropped DLLs.
## MITRE ATT&CK Techniques

- [T1574](https://attack.mitre.org/techniques/T1574)
- [T1574.002](https://attack.mitre.org/techniques/T1574/002)

## License

- `Elastic License v2`
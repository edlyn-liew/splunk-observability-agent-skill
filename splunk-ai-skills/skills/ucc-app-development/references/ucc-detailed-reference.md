# UCC Generator Detailed Reference

## Official Documentation Links

### UCC Framework
- [UCC Generator Docs (Main)](https://splunk.github.io/addonfactory-ucc-generator/)
- [Quickstart Guide](https://splunk.github.io/addonfactory-ucc-generator/quickstart/)
- [Configuration Reference](https://splunk.github.io/addonfactory-ucc-generator/configurations/)
- [Dashboard Generation](https://splunk.github.io/addonfactory-ucc-generator/dashboard/)
- [OpenAPI Generation](https://splunk.github.io/addonfactory-ucc-generator/openapi/)
- [GitHub Repository](https://github.com/splunk/addonfactory-ucc-generator)
- [PyPI Package](https://pypi.org/project/splunk-add-on-ucc-framework/)
- [npm Package](https://www.npmjs.com/package/@splunk/add-on-ucc-framework)

### Splunk Development
- [Packaging Toolkit & app.manifest](https://dev.splunk.com/enterprise/reference/packagingtoolkit/pkgtoolkitappmanifest)
- [Python SDK](https://dev.splunk.com/enterprise/docs/devtools/python/sdk-python)
- [CIM Data Models](https://help.splunk.com/en/data-management/common-information-model/6.3)
- [Add-on Sourcetypes Overview](https://docs.splunk.com/Documentation/AddOns/released/Overview/Sourcetypes)

## Quick Reference

### globalConfig.json Structure
```json
{
  "meta": {"name": "addon_name", "restRoot": "addon_name", "version": "1.0.0", "displayName": "Display Name", "schemaVersion": "0.0.3"},
  "pages": {
    "configuration": {"title": "Configuration", "tabs": []},
    "inputs": {"title": "Inputs", "table": {"header": [], "actions": ["edit","delete","clone","enable"]}, "services": []},
    "dashboard": {"panels": []}
  }
}
```

### Entity Field Properties
type, field, label, help, required, encrypted, defaultValue, validators, options, modifyFieldsOnValue

### Modular Input Template

```python
import import_declare_test
import sys, json
from splunklib import modularinput as smi

class MyInput(smi.Script):
    def get_scheme(self):
        scheme = smi.Scheme("My Input")
        scheme.use_external_validation = True
        scheme.use_single_instance = False
        scheme.add_argument(smi.Argument(name="api_endpoint", title="API Endpoint", required_on_create=True))
        return scheme

    def validate_input(self, definition):
        if not definition.parameters["api_endpoint"].startswith("https://"):
            raise ValueError("Must use HTTPS")

    def stream_events(self, inputs, ew):
        for input_name, input_item in inputs.inputs.items():
            try:
                event = smi.Event()
                event.stanza = input_name
                event.data = json.dumps({"status": "ok"})
                ew.write_event(event)
            except Exception as e:
                ew.log("ERROR", f"Error in {input_name}: {str(e)}")

if __name__ == "__main__":
    sys.exit(MyInput().run(sys.argv))
```

### Alert Action Template

```python
import import_declare_test
from splunktaucclib.alert_actions_base import ModularAlertBase

class AlertAction(ModularAlertBase):
    def __init__(self, ta_name, alert_name):
        super().__init__(ta_name, alert_name)

    def validate_params(self):
        if not self.get_param("webhook_url"):
            self.log_error("webhook_url is required")
            return False
        return True

    def process_event(self, helper, *args, **kwargs):
        for event in helper.get_events():
            self.log_info(f"Processing: {event}")
        return 0

if __name__ == "__main__":
    AlertAction("my_addon", "my_alert").execute()
```

### Troubleshooting
1. **ModuleNotFoundError** → Check `package/lib/requirements.txt`
2. **Input runs once** → Set `use_single_instance = False`
3. **urllib3 SSL error** → Pin `urllib3 < 2`
4. **UI not loading** → Check browser console; verify `appserver/static/`
5. **REST 500** → `index=_internal source=*splunkd* ERROR addon_name`
6. **OAuth popup blocked** → Verify popup size and redirect URL

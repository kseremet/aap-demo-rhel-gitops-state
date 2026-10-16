
n## Increase TCP Listen Backlog

To enhance the incoming connection handling, update the `net.core.somaxconn` parameter in the system settings. Here’s how to do it:

```bash
sudo sysctl -w net.core.somaxconn=128
```
Make sure to test the configuration after applying it to verify effective changes.
# Example playbooks

## Setup virtual environment
``` bash
python3 -m venv .venv
.venv/bin/python3 -m pip install -U ansible pytest
```
## Setup to run ansible
``` bash
mkdir -p library
ln -s ../json_patch.py library/json_patch.py
```
## Examples

### append-if-not-exist
Shows an example of how to test and add values to a list if they are not already there.

``` bash
.venv/bin/ansible-playbook -vvv -i local, --connection=local examples/append-if-not-exist.yaml

```

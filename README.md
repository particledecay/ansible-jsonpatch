# ansible-jsonpatch
An Ansible module for patching JSON files

## Description
This Ansible module follows [RFC 6901](https://tools.ietf.org/html/rfc6901) for JSON notation and [RFC 6902](https://tools.ietf.org/html/rfc6902) for JSON patching.

In addition to the standard, there is an extra notation feature not outlined in RFC 6901: you can specify *every* member of an array with a `*` character in the path.

## Installation
In order to make use of this module, download the [json_patch.py](https://raw.githubusercontent.com/ParticleDecay/ansible-jsonpatch/master/json_patch.py) file to the `library` directory in the root of your Ansible project:
```
myproject/
├── ansible.cfg
├── inv/
├── library/
│   ├── json_patch.py
├── playbooks/
├── roles/
```

## Usage
The module `json_patch` takes the following options:

| Name    | Description     | Choices |
| ------- | --------------- | ------- |
| `src` | The path to a file containing JSON | |
| `dest` | The path to an optional output file to write the patched JSON (default is to overwrite `src` file) | |
| `operations` | A list of valid operations to perform against the given JSON file | See [RFC 6902](https://tools.ietf.org/html/rfc6902#section-4) |
| `backup` | Backup the JSON file (the one that will be overwritten) before editing (default: `no`) | `yes`/`no` |
| `unsafe_writes` | Allow Ansible to fall back to unsafe (i.e. non-atomic) methods of writing files (default: `no`) | `yes`/`no` |
| `pretty` | Write JSON output in human-readable format | `yes`/`no` |
| `create` | Create file if it does not exist | `yes`/`no` |
| `create_type` | Type of JSON structure to create if empty (default: `object`) | `object`/`array` |

#### Operations
Operations may consist of two or more of the following properties:

| Name | Required | Value |
| ---- | -------- | ----- |
| `op` | yes | one of `add`, `remove`, `replace`, `move`, `copy`, or `test` |
| `path` | yes | RFC 6901 JSON notation |
| `from` | no | RFC 6901 JSON notation (only applies to `move` and `copy`)|
| `value` | no | any valid JSON value |

#### Special Notations
There are two special notations:
  - The `-` character refers to the *end* of an array (for appending a non-existent value)
  - (non-RFC 6901 compliant) The `*` character may refer to *every* member of an array (for testing each member)

## Examples

### ADD
##### Test JSON
```json
{
    "foo": "bar"
}
```
##### Patch
```yaml
- name: Add or replace 'baz' member
  json_patch:
    src: "test.json"
    operations:
      - op: add
        path: "/baz"
        value: "quz"
```
##### Patched JSON
```json
{
    "foo": "bar",
    "baz": "quz"
}
```

### REMOVE
##### Test JSON
```json
[
    {"foo": "bar", "baz": "quz"},
    {"foo": "qux", "hello": "world"}
]
```
##### Patch
```yaml
- name: Remove the 'hello' property
  json_patch:
    src: "test.json"
    operations:
      - op: remove
        path: "/1/hello"
```
##### Patched JSON
```json
[
    {"foo": "bar", "baz": "quz"},
    {"foo": "qux"}
]
```

### REPLACE
##### Test JSON
```json
[
    {
        "foo": [
            {"vegetable": "corn"},
            {"vegetable": "broccoli"},
            {"vegetable": "tomato"}
        ]
    }
]
```
##### Patch
```yaml
- name: Ensure 'tomato' isn't listed as a vegetable
  json_patch:
    src: "test.json"
    operations:
      - op: replace
        path: "/0/foo/2/vegetable"
        value: "cauliflower"
```
##### Patched JSON
```json
[
    {
        "foo": [
            {"vegetable": "corn"},
            {"vegetable": "broccoli"},
            {"vegetable": "cauliflower"}
        ]
    }
]
```

### MOVE
##### Test JSON
```json
[
    {"one": 1, "two": 2, "six": 6},
    {"four": 4, "five": 5, "three": 3}
]
```
##### Patch
```yaml
- name: Move the numbers into their appropriate objects
  json_patch:
    src: "test.json"
    operations:
      - op: move
        from: "/0/six"
        path: "/1/six"
      - op: move
        from: "/1/three"
        path: "/0/three"
```
##### Patched JSON
```json
[
    {"one": 1, "two": 2, "three": 3},
    {"four": 4, "five": 5, "six": 6}
]
```

### COPY
##### Test JSON
```json
[
    {"name": "The Shawshank Redemption", "rating": "good"},
    {"name": "The Dark Knight", "rating": "bad"}
]
```
##### Patch
```yaml
- name: Correct the rating for The Dark Knight
  json_patch:
    src: "test.json"
    backup: yes
    operations:
      - op: copy
        from: "/0/rating"
        path: "/1/rating"
```
##### Patched JSON
```json
[
    {"name": "The Shawshank Redemption", "rating": "good"},
    {"name": "The Dark Knight", "rating": "good"}
]
```

### TEST
##### Test JSON
```json
{
    "name": "Ansible",
    "gitPath": "/home/ubuntu/projects/ansible",
    "enabled": true
}
```
##### Patch
```yaml
- name: Ensure the Ansible project is enabled
  json_patch:
    src: "test.json"
    operations:
      - op: test
        path: "/enabled"
        value: true
```

### SPECIAL NOTATIONS
#### End of array character (`-`)
##### Test JSON
```json
{
    "media": "/mnt/media",
    "scan": [
      {"movie": "Independence Day"},
      {"movie": "The Sandlot"},
      {"movie": "Bird Box"}
    ]
}
```
##### Patch
```yaml
- name: Add all of our movies to the list
  json_patch:
    src: "test.json"
    operations:
      - op: add
        path: "/scan/-"
        value:
          movie: "{{ item }}"
  with_items:
    - "Se7en"
    - "Firestarter"
    - "The Last Samurai"
    - "Fast and the Furious 17"
```
##### Patched JSON
```json
{
    "media": "/mnt/media",
    "scan": [
      {"movie": "Independence Day"},
      {"movie": "The Sandlot"},
      {"movie": "Bird Box"},
      {"movie": "Se7en"},
      {"movie": "Firestarter"},
      {"movie": "The Last Samurai"},
      {"movie": "Fast and the Furious 17"}
    ]
}
```

#### Each array member character (`*`)
##### Test JSON
```json
{
    "media": "/mnt/media",
    "scan": [
      {"movie": "Independence Day"},
      {"movie": "The Sandlot"},
      {"movie": "Bird Box"},
      {"movie": "Se7en"},
      {"movie": "Firestarter"},
      {"movie": "The Last Samurai"},
      {"movie": "Fast and the Furious 17"}
    ]
}
```
##### Patch
```yaml
- name: Ensure The Last Samurai is definitely in our list
  json_patch:
    src: "test.json"
    operations:
      - op: test
        path: "/scan/*/movie"
        value: "The Last Samurai"
```

### Examples
For more examples of how to use this module see `exmaples` folder.

## Testing

### create virtual environment
``` bash
python3 -m venv .venv
.venv/bin/python3 -m pip install -U ansible pytest
```
Testing is simple once you have pytest installed:
```bash
.venv/bin/py.test -v
```
Note you may have to copy the `json_patch.py` file into your Ansible installation's `lib/ansible/modules/files/` dir.

## Contributing
PRs welcome! Just make sure you update tests as well.

Current needs:
- Special character handling ([RFC 6901 - Syntax](https://tools.ietf.org/html/rfc6901#section-3))
- Applying `*` support to other operations:
  - `add` should affect every member of an array
  - `remove` should remove the path from each member of an array
  - `replace` should replace for each member of an array

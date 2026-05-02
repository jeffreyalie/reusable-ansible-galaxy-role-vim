# vim

Ansible role to install `vim` on Ubuntu systems.

## Requirements

None. Uses only `ansible.builtin` modules.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `vim_package` | `vim` | Package name to install |
| `vim_state` | `present` | `present` to install, `absent` to remove |

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: jeffrey.vim
```

## License

MIT

## Author

jeffrey

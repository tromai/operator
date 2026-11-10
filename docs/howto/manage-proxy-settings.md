(manage-proxy-configs)=
# How to manage proxy configs

Juju users can set the `juju-http-proxy`, `juju-https-proxy`, and `juju-no-proxy` {external+juju:ref}`model configuration keys <list-of-model-configuration-keys>` to influence how charm or charm workload makes any HTTP request. A {ref}`mature charm <charm-maturity>` must respect these proxy configs.

## Access the `juju-*-proxy` model configs

Use [](ops.Model.proxy_config) to get the model's proxy configs. This returns an [](ops.ProxyConfig) instance:

```python
class MyCharm(ops.CharmBase):
    def _on_pebble_ready(self, event: ops.PebbleReadyEvent):
        proxy = self.model.proxy_config
        environment = {}
        if proxy.http:
            environment['HTTP_PROXY'] = proxy.http
        if proxy.https:
            environment['HTTPS_PROXY'] = proxy.https
        if proxy.no:
            environment['NO_PROXY'] = ','.join(proxy.no)
        ...
```

[](ops.ProxyConfig) stores empty strings for `juju-http-proxy` and `juju-https-proxy` as `None`.

[](ops.ProxyConfig)'s `no` attribute contains the comma-separated entries of `juju-no-proxy`. An empty `juju-no-proxy` is read as an empty tuple. Entries in `no` can contain wildcards (`*.example.com`, or `*` for everything), domain suffixes (`.example.com`), or CIDRs (`10.0.0.0/8`).

Setting a model config from the Juju CLI doesn't fire any event. The charm can only read the latest value the next time an event fire. `ops` exposes those model configs as plain strings, and doesn't validate them.

## Manage proxy options

The charm and its workload are separate processes, and each one must respect the proxy configs. Both read the values from [](ops.Model.proxy_config), but they receive them in different ways:

- The charm makes its own HTTP requests, for example to download a file.
- The workload makes its own HTTP requests. How it reads proxy configs depends on the workload. Most workloads read the `*_proxy` or `*_PROXY` environment variables. Some read a configuration file or start-up flags (for example, [`trino-k8s-operator`](https://github.com/canonical/trino-k8s-operator) uses JVM flags).

Because Juju doesn't fire an event when the proxy configs change, apply them in a handler that runs on every event, not only at install time or when the workload is first started. Compare the new value with what the workload already uses, and restart the workload only when the value changed. This is the approach that [`cos-coordinated-workers`](https://github.com/canonical/cos-coordinated-workers) uses: it rebuilds the Pebble layer on every event and restarts the services if the layer differs from the current plan.

Both the charm and the workload can use this helper to turn [](ops.ProxyConfig) into environment variables:

```python
def proxy_environment(proxy: ops.ProxyConfig) -> dict[str, str]:
    """Return the proxy environment variables for the charm or the workload."""
    environment: dict[str, str] = {}
    for name, url in (('HTTP_PROXY', proxy.http), ('HTTPS_PROXY', proxy.https)):
        if not url:
            continue
        # Set both lower and upper env variables, because tools might expects either.
        environment[name] = environment[name.lower()] = url
    if environment:
        no_proxy = ','.join(proxy.no)
        environment['NO_PROXY'] = environment['no_proxy'] = no_proxy
    return environment
```

Keep the output of the helper stable. The same input must always give the same dictionary, in the same order. Otherwise, the charm sees a change on every event and restarts the workload each time.

### Manage proxy options for the charm

The charm process is short-lived: Juju starts a new one for every event. Set the environment variables at the start of the handler, before the charm makes a request. Libraries such as {external+python:func}`urllib.request.urlopen` and `requests` detect the proxy configs from these variables:

```python
import os
import urllib.request


class MyCharm(ops.CharmBase):
    def get_some_value_from_github_api(self):
        os.environ.update(proxy_environment(self.model.proxy_config))
        with urllib.request.urlopen('https://api.github.com') as response:
            return response.read().decode('utf-8')
```

To avoid changing the environment, pass the values to the library instead. For example, pass a {external+python:class}`urllib.request.ProxyHandler` to {external+python:func}`urllib.request.build_opener`. `urllib` only reads `no_proxy` from the environment. For `requests`, see [Advanced usage | Proxies](https://requests.readthedocs.io/en/latest/user/advanced/#proxies).

### Manage proxy options for the workload on Kubernetes

Pebble services don't inherit the charm's environment. Pass the proxy configs in the `environment` of each service in the Pebble layer. When the layer changes, `Container.replan()` restarts only the services whose configuration changed. If the layer is the same, `replan()` leaves the running services alone, so it's safe to call on every event.

```python
class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        framework.observe(self.on['workload'].pebble_ready, self._update_workload)
        framework.observe(self.on.config_changed, self._update_workload)
        framework.observe(self.on.update_status, self._update_workload)

    def _update_workload(self, event: ops.EventBase) -> None:
        container = self.unit.get_container('workload')
        if not container.can_connect():
            self.unit.status = ops.WaitingStatus('waiting for Pebble')
            return
        try:
            environment = proxy_environment(self.model.proxy_config)
        except ValueError as e:
            self.unit.status = ops.BlockedStatus(str(e))
            return
        container.add_layer(
            'workload',
            {
                'services': {
                    'my-service': {
                        'override': 'replace',
                        'command': '/bin/my-service',
                        'startup': 'enabled',
                        'environment': environment,
                    }
                }
            },
            combine=True,
        )
        container.replan()
        self.unit.status = ops.ActiveStatus()
```

Observing `update-status` makes the charm pick up a change within the update-status interval (five minutes by default). Observe any other event that your charm handles, to pick up the change sooner.

If the workload reads the proxy configs from a file or from start-up flags, build the file content or the `command` from the same values. Put them in the layer, or push the file with `Container.push()` and restart the service when the content changed.

### Manage proxy options for the workload on a machine

A machine charm must write the values to the place where the workload reads them, and restart or reload the workload when they change. For a workload that runs as a systemd service, a drop-in file with `Environment=` lines is a common choice. Only write the file and restart the service when the content has changed:

```python
import pathlib
import subprocess

DROP_IN = pathlib.Path('/etc/systemd/system/my-service.service.d/proxy.conf')


def write_proxy_drop_in(environment: dict[str, str]) -> bool:
    """Write the systemd drop-in file for the service, and return whether it changed."""
    content = None
    if environment:
        # systemd expands % specifiers in values, so escape them.
        lines = [f'Environment="{k}={v.replace("%", "%%")}"' for k, v in sorted(environment.items())]
        content = '[Service]\n' + '\n'.join(lines) + '\n'
    current = DROP_IN.read_text() if DROP_IN.exists() else None
    if content == current:
        return False
    if content is None:
        DROP_IN.unlink()
    else:
        DROP_IN.parent.mkdir(parents=True, exist_ok=True)
        # The URL can contain credentials, so restrict who can read the file.
        DROP_IN.touch(mode=0o600)
        DROP_IN.write_text(content)
    return True


class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        framework.observe(self.on.config_changed, self._update_workload)
        framework.observe(self.on.update_status, self._update_workload)

    def _update_workload(self, event: ops.EventBase) -> None:
        if write_proxy_drop_in(proxy_environment(self.model.proxy_config)):
            subprocess.run(['/usr/bin/systemctl', 'daemon-reload'], check=True, capture_output=True)
            subprocess.run(
                ['/usr/bin/systemctl', 'restart', 'my-service'], check=True, capture_output=True
            )
```

Keep the systemd calls in your workload module, as described in {ref}`run-workloads-with-a-charm-machines`. If the workload reads a configuration file, render the proxy settings into that file in the same way, and reload the workload when the content changed. For a snap, use `snap set` or the snap's own configuration instead of a drop-in file.

```{admonition} Best practice
:class: hint

Don't include the proxy URL in log messages or in the status message, as it can contain credentials.
Set both lowercase and uppercase proxy environment variables, as tools can depend on either of them.
Each tool receives proxy config differently, so check how your workload reads them.
```

Examples: [`cos-coordinated-workers`](https://github.com/canonical/cos-coordinated-workers) and [`mysql-operators`](https://github.com/canonical/mysql-operators) pass the configs to workloads through the Pebble layer. [`trino-k8s-operator`](https://github.com/canonical/trino-k8s-operator) validates and converts the values to JVM options.

## Test the feature

### Write unit tests

See first: {ref}`write-unit-tests-for-a-charm`

In your unit tests, use [](ops.testing.ProxyConfig) and the `proxy_config` attribute of [](ops.testing.Model) to set the model's proxy settings in the input `State`. Then run an event, and check what your charm did with the values.

For example:

```python
from ops import testing

from charm import MyCharm


def run(proxy_config: testing.ProxyConfig | None) -> testing.State:
    ctx = testing.Context(MyCharm)
    container = testing.Container('workload', can_connect=True)
    state_in = testing.State(
        containers={container},
        model=testing.Model(proxy_config=proxy_config),
    )
    return ctx.run(ctx.on.pebble_ready(container), state_in)


def environment(state: testing.State) -> dict[str, str]:
    return state.get_container('workload').plan.services['my-service'].environment


def test_proxy_settings_reach_the_workload():
    state_out = run(
        testing.ProxyConfig(
            https='http://proxy.example.com:3128',
            no=('127.0.0.1', 'localhost', '::1', '10.0.0.0/8'),
        )
    )
    env = environment(state_out)
    assert env['HTTPS_PROXY'] == env['https_proxy'] == 'http://proxy.example.com:3128'
    assert 'HTTP_PROXY' not in env
    assert env['NO_PROXY'] == '127.0.0.1,localhost,::1,10.0.0.0/8,.svc,.svc.cluster.local'
    assert state_out.unit_status == testing.ActiveStatus()


def test_no_proxy_configured():
    # Both None and the default ProxyConfig() mean that the user hasn't set a proxy.
    for proxy_config in (None, testing.ProxyConfig()):
        state_out = run(proxy_config)
        assert environment(state_out) == {}
        assert state_out.unit_status == testing.ActiveStatus()
```

`testing.ProxyConfig` has the same attributes as `ops.ProxyConfig`, and it uses Juju's defaults for `juju-http*-proxy` and `juju-no-proxy` model configs.

`Model.proxy_config` is `None` by default.

(manage-proxy-settings-integration-tests)=
### Write integration tests

See first: {ref}`write-integration-tests-for-a-charm`

In an integration test, change the proxy settings with [`juju.model_config`](jubilant.Juju.model_config), and check that the workload has picked up the values.

Because Juju doesn't emit an event when the model configuration changes, the charm only notices the change when the next event runs. The example charm above observes `update-status`, so the test sets `update-status-hook-interval` to a short interval instead of waiting for the default of five minutes. Then the test waits for the workload to change. Waiting for active status isn't enough, because the unit is already active, so the wait condition also checks the workload.

For example, in `tests/integration/test_proxy.py`. This test is for a Kubernetes charm, and reads the Pebble plan. For a machine charm, replace `workload_plan` with a command that shows the environment of your service, such as `juju.ssh(unit, 'systemctl', 'show', 'my-service', '--property=Environment')`.

```python
import pathlib

import jubilant
import pytest

PROXY_URL = 'http://proxy.example.com:3128'
UNIT = 'your-app/0'


@pytest.fixture
def fast_update_status(juju: jubilant.Juju):
    """Make Juju run update-status every 10 seconds, so the charm notices model config changes."""
    juju.model_config({'update-status-hook-interval': '10s'})
    yield
    juju.model_config(reset='update-status-hook-interval')


def workload_plan(juju: jubilant.Juju) -> str:
    return juju.ssh(UNIT, '/charm/bin/pebble', 'plan', container='workload')


def test_proxy_settings_reach_the_workload(
    charm: pathlib.Path, juju: jubilant.Juju, fast_update_status: None
):
    try:
        juju.model_config({
            'juju-https-proxy': PROXY_URL,
            'juju-no-proxy': '127.0.0.1,localhost,::1,10.0.0.0/8',
        })
        juju.wait(
            lambda status: (
                jubilant.all_active(status) and f'HTTPS_PROXY: {PROXY_URL}' in workload_plan(juju)
            ),
            timeout=120,
        )
    finally:
        juju.model_config(reset=['juju-https-proxy', 'juju-no-proxy'])
    # Removing the proxy settings removes the variables.
    juju.wait(
        lambda status: jubilant.all_active(status) and 'HTTPS_PROXY' not in workload_plan(juju),
        timeout=120,
    )


def test_invalid_proxy_blocks_the_unit(
    charm: pathlib.Path, juju: jubilant.Juju, fast_update_status: None
):
    try:
        juju.model_config({'juju-https-proxy': 'not-a-url'})
        juju.wait(jubilant.all_blocked, timeout=120)
    finally:
        juju.model_config(reset='juju-https-proxy')
    juju.wait(jubilant.all_active, timeout=120)
```

This example assumes that an earlier test deployed the charm, as in {ref}`write-integration-tests-for-a-charm`. We recommend including the `charm` fixture (even though it's not used) so that the test fails immediately if a `.charm` file isn't available.

See also: [](jubilant.Juju.model_config)

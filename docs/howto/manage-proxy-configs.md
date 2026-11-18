(manage-proxy-configs)=
# How to manage proxy configs

Juju users can set the `juju-http-proxy`, `juju-https-proxy`, and `juju-no-proxy` {external+juju:ref}`model configs <list-of-model-configuration-keys>` to control how a charm and its workload make HTTP requests. A {ref}`mature charm <charm-maturity>` must respect these proxy configs.

## Read the proxy configs

Use [](ops.Model.proxy_config) to get the model's proxy configs. This returns an [](ops.ProxyConfig) instance:

```python
import ops


class MyCharm(ops.CharmBase):
    def _on_pebble_ready(self, event: ops.PebbleReadyEvent):
        proxy = self.model.proxy_config
        environment = {}
        if proxy.http:
            environment['HTTP_PROXY'] = proxy.http
            environment['http_proxy'] = proxy.http
        if proxy.https:
            environment['HTTPS_PROXY'] = proxy.https
            environment['https_proxy'] = proxy.https
        if proxy.no:
            environment['NO_PROXY'] = ','.join(proxy.no)
            environment['no_proxy'] = ','.join(proxy.no)
        ...
```

[](ops.ProxyConfig) stores an empty `juju-http-proxy` or `juju-https-proxy` as `None`.

The `no` attribute of [](ops.ProxyConfig) contains the comma-separated entries of `juju-no-proxy` as a tuple of strings. An empty `juju-no-proxy` is an empty tuple. Entries in `no` can contain wildcards (`*.example.com`, or `*` for everything), domain suffixes (`.example.com`), or CIDRs (`10.0.0.0/8`).

Ops passes the proxy configs on as plain strings, and doesn't validate them. Juju doesn't fire an event when the proxy configs change, so the charm must read the proxy configs in every event that it observes, following the {ref}`reconciler pattern <handling-events-holistically>`:

```python
import ops


class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        self.container = self.unit.get_container('proxy-app')
        framework.observe(self.on['proxy-app'].pebble_ready, self._reconcile)
        framework.observe(self.on.config_changed, self._reconcile)
        framework.observe(self.on.update_status, self._reconcile)

    def _proxy_env(self) -> dict[str, str]:
        proxy = self.model.proxy_config
        # Sort the entries, so that reordering juju-no-proxy doesn't restart the workload.
        no_proxy = ','.join(sorted(proxy.no))
        return {
            # Use an empty string for an unset proxy config, so that the workload
            # stops using a value that the charm set in an earlier event.
            'HTTP_PROXY': proxy.http or '',
            'HTTPS_PROXY': proxy.https or '',
            'NO_PROXY': no_proxy,
            'http_proxy': proxy.http or '',
            'https_proxy': proxy.https or '',
            'no_proxy': no_proxy,
        }

    def _reconcile(self, event: ops.EventBase) -> None:
        if not self.container.can_connect():
            return
        proxy_env = self._proxy_env()
        # Pass proxy_env to the workload, as described in the next section.
        ...
```

## Manage proxy configs for the workload

The workload typically reads the proxy configs itself, so the charm needs to put the proxy configs where the workload expects them.

Most workloads read the `*_proxy` or `*_PROXY` environment variables. Some read a configuration file or start-up flags. For example, [`trino-k8s-operator`](https://github.com/canonical/trino-k8s-operator) passes the proxy configs to its workload as JVM flags.

### Manage proxy configs for the workload on Kubernetes

When the workload runs in a sidecar container, the charm can pass the proxy configs with Pebble.

If the workload reads environment variables, create a Pebble layer that merges the variables into the existing service definition:

```python
class MyCharm(ops.CharmBase):
    ...

    def _proxy_layer(self, proxy_env: dict[str, str]) -> ops.pebble.Layer:
        return ops.pebble.Layer({
            'services': {
                'proxy-app': {
                    'override': 'merge',
                    'environment': proxy_env,
                },
            },
        })
```

The layer only adds environment variables to the `proxy-app` service, so add it after the layer that defines the service.

In the reconciler, add the layer and call [](ops.Container.replan). Pebble only restarts a service if its configuration changed, so the charm can add the layer and replan in every event, and the workload only restarts when the proxy configs change:

```python
class MyCharm(ops.CharmBase):
    ...

    def _reconcile(self, event: ops.EventBase) -> None:
        if not self.container.can_connect():
            return
        proxy_layer = self._proxy_layer(self._proxy_env())
        self.container.add_layer('proxy-workload', proxy_layer, combine=True)
        self.container.replan()
```

[`cos-coordinated-workers`](https://github.com/canonical/cos-coordinated-workers/blob/main/src/coordinated_workers/worker.py) uses a similar approach: it rebuilds the Pebble layer in every event that it reconciles, and restarts the services if the layer differs from the current plan.

If the workload reads a configuration file, write the proxy configs to a file in the workload container. See {ref}`files-in-containers`.

(manage-proxy-configs-machine)=
### Manage proxy configs for the workload in a machine charm

In a machine charm, the charm and its workload run on the same machine, so the charm can write the proxy configs directly to where the workload reads them.

A long-running workload, such as a systemd service or a snap, doesn't inherit the environment of the charm process. The charm must store the proxy configs somewhere that persists between events, then restart the workload so that it reads the new values.

How the charm writes the proxy configs depends on how the workload reads them:

| How the workload reads the proxy configs | How the charm writes the proxy configs |
|---|---|
| Environment variables, in a systemd service | Write a systemd drop-in file in `/etc/systemd/system/<service>.service.d/` with `Environment=` lines, using {external+charmlibs:ref}`pathops <charmlibs-pathops>`. Then reload systemd and restart the service with {external+charmlibs:ref}`charmlibs-systemd <charmlibs-systemd>`. |
| Environment variables, in a command that the charm runs, such as `git` or `curl` | Pass the variables with the `env` argument of {external+python:func}`subprocess.run`. |
| A configuration file | Render the workload's configuration to a local file with {external+charmlibs:ref}`pathops <charmlibs-pathops>`. |
| A configuration API, such as snap options, or the workload's own CLI options | Set snap options with {external+charmlibs:ref}`charmlibs-snap <charmlibs-snap>`, or call the workload's API or tool. |

The reconciler is similar to a Kubernetes charm: convert [](ops.ProxyConfig) into the format that the workload expects, compare the result with what the workload has, and restart the workload only if they differ. For example, for a systemd service that reads environment variables:

```python
import ops
from charmlibs import pathops, systemd

SERVICE = 'my-workload.service'
PROXY_DROP_IN = pathops.LocalPath(f'/etc/systemd/system/{SERVICE}.d/proxy.conf')


class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        framework.observe(self.on.config_changed, self._reconcile)
        framework.observe(self.on.update_status, self._reconcile)

    def _proxy_env(self) -> dict[str, str]:
        ...  # The same as in "Read the proxy configs".

    def _reconcile(self, event: ops.EventBase) -> None:
        lines = ['[Service]']
        for name, value in self._proxy_env().items():
            # systemd expands '%' specifiers in Environment=, and proxy URLs
            # can contain '%' in percent-encoded credentials.
            value = value.replace('%', '%%')
            lines.append(f'Environment="{name}={value}"')
        # ensure_contents only writes the file if the contents differ.
        if pathops.ensure_contents(PROXY_DROP_IN, '\n'.join(lines) + '\n'):
            systemd.daemon_reload()
            systemd.service_restart(SERVICE)
```

These machine charms pass the proxy configs to their workloads:

- [`k8s`](https://github.com/canonical/k8s-operator/tree/main/charms/worker/k8s/src/charm.py): writes a systemd drop-in file for the `containerd` snap service.
- [`vault`](https://github.com/canonical/vault-k8s-operator/tree/main/machine/src/charm.py): writes a systemd drop-in file for the Vault snap service.
- [`lxd`](https://github.com/canonical/charm-lxd/tree/main/src/charm.py): sets the LXD `core.proxy_http`, `core.proxy_https`, and `core.proxy_ignore_hosts` options through the LXD API.

## Manage proxy configs for the charm process

The charm can make its own HTTP requests, for example to download a file or call an API.

Juju passes the proxy configs to the charm only as the `JUJU_CHARM_HTTP_PROXY`, `JUJU_CHARM_HTTPS_PROXY`, and `JUJU_CHARM_NO_PROXY` environment variables. HTTP libraries such as {external+python:mod}`urllib.request` and `requests` read the `*_proxy` environment variables instead, so by default they ignore the proxy configs.

Pass the proxy configs to the library when the charm makes a request. For example, pass a {external+python:class}`urllib.request.ProxyHandler` to {external+python:func}`urllib.request.build_opener`:

```python
import os
import urllib.request

import ops


class MyCharm(ops.CharmBase):
    def _get_latest_release(self) -> str:
        proxy = self.model.proxy_config
        proxies = {}
        if proxy.http:
            proxies['http'] = proxy.http
        if proxy.https:
            proxies['https'] = proxy.https
        # ProxyHandler has no argument for juju-no-proxy, and reads
        # the no_proxy environment variable instead.
        os.environ['no_proxy'] = ','.join(proxy.no)
        opener = urllib.request.build_opener(urllib.request.ProxyHandler(proxies))
        url = 'https://api.github.com/repos/canonical/operator/releases/latest'
        with opener.open(url) as response:
            return response.read().decode('utf-8')
```

For `requests`, see [requests | Advanced usage | Proxies](https://requests.readthedocs.io/en/latest/user/advanced/#proxies).

## Best practices

- Don't include the proxy URL in log messages or in the status message, as it can contain credentials.
- Set both lowercase and uppercase proxy environment variables, as tools can depend on either of them.
- Tools accept the proxy configs in different formats, especially `no_proxy`. Convert and validate the proxy configs for each tool that the charm passes them to.

## Test the feature

### Write unit tests

See first: {ref}`write-unit-tests-for-a-charm`

To set the proxy configs in a unit test, pass a [](ops.testing.ProxyConfig) to [](ops.testing.Model). The charm reads the values with [](ops.Model.proxy_config), as it would in a real Juju model. A default `ProxyConfig()` matches a model with Juju's default proxy configs: no HTTP or HTTPS proxy, and `juju-no-proxy` set to `127.0.0.1,localhost,::1`.

Test that the proxy configs reach the workload, and that the charm clears them when the proxy configs are removed.

#### Test a Kubernetes charm

For a Kubernetes charm, check the environment of the workload service in the Pebble plan:

```python
import ops
from ops import testing

from charm import MyCharm

PROXY_URL = 'http://proxy.example.com:3128'


def workload_layer() -> ops.pebble.Layer:
    """The layer that defines the workload service, before any proxy configs."""
    return ops.pebble.Layer({
        'services': {
            'proxy-app': {
                'override': 'replace',
                'command': 'my-workload',
                'startup': 'enabled',
            }
        }
    })


def test_proxy_configs_reach_the_workload():
    ctx = testing.Context(MyCharm)
    container = testing.Container(
        'proxy-app',
        can_connect=True,
        layers={'base': workload_layer()},
    )
    proxy_config = testing.ProxyConfig(
        https=PROXY_URL,
        no=('localhost', '10.0.0.0/8'),
    )
    state_in = testing.State(
        containers={container},
        model=testing.Model(proxy_config=proxy_config),
    )

    state_out = ctx.run(ctx.on.pebble_ready(container), state_in)

    plan = state_out.get_container('proxy-app').plan
    environment = plan.services['proxy-app'].environment
    assert environment['HTTPS_PROXY'] == PROXY_URL
    assert environment['https_proxy'] == PROXY_URL
    assert environment['HTTP_PROXY'] == ''
    assert environment['NO_PROXY'] == '10.0.0.0/8,localhost'


def test_removed_proxy_configs_are_cleared():
    ctx = testing.Context(MyCharm)
    # The workload has proxy configs from an earlier event.
    proxy_layer = ops.pebble.Layer({
        'services': {
            'proxy-app': {
                'override': 'merge',
                'environment': {'HTTPS_PROXY': PROXY_URL, 'https_proxy': PROXY_URL},
            }
        }
    })
    container = testing.Container(
        'proxy-app',
        can_connect=True,
        layers={'base': workload_layer(), 'proxy-workload': proxy_layer},
    )
    state_in = testing.State(
        containers={container},
        model=testing.Model(proxy_config=testing.ProxyConfig()),
    )

    state_out = ctx.run(ctx.on.update_status(), state_in)

    plan = state_out.get_container('proxy-app').plan
    environment = plan.services['proxy-app'].environment
    assert environment['HTTPS_PROXY'] == ''
    assert environment['https_proxy'] == ''
```

#### Test a machine charm

For a machine charm, check the file or API that the charm writes the proxy configs to. Replace the parts of the charm that change the machine, so that the tests don't need root access or a real service.

The following tests are for the machine charm in {ref}`Manage proxy configs for the workload in a machine charm <manage-proxy-configs-machine>`:

```python
import pathlib

import pytest
from charmlibs import pathops
from ops import testing

import charm

PROXY_URL = 'http://proxy.example.com:3128'


@pytest.fixture
def drop_in(monkeypatch: pytest.MonkeyPatch, tmp_path: pathlib.Path) -> pathops.LocalPath:
    """Write the drop-in file to a temporary directory instead of /etc/systemd."""
    path = pathops.LocalPath(tmp_path / 'proxy.conf')
    monkeypatch.setattr(charm, 'PROXY_DROP_IN', path)
    return path


@pytest.fixture
def restarted(monkeypatch: pytest.MonkeyPatch) -> list[str]:
    """Record the services that the charm restarts, instead of calling systemctl."""
    services: list[str] = []

    def service_restart(*args: str) -> bool:
        services.extend(args)
        return True

    monkeypatch.setattr(charm.systemd, 'daemon_reload', lambda: True)
    monkeypatch.setattr(charm.systemd, 'service_restart', service_restart)
    return services


def test_proxy_configs_reach_the_workload(drop_in: pathops.LocalPath, restarted: list[str]):
    ctx = testing.Context(charm.MyCharm)
    proxy_config = testing.ProxyConfig(https=PROXY_URL, no=('localhost', '10.0.0.0/8'))
    state_in = testing.State(model=testing.Model(proxy_config=proxy_config))

    ctx.run(ctx.on.config_changed(), state_in)

    content = drop_in.read_text()
    assert f'Environment="HTTPS_PROXY={PROXY_URL}"' in content
    assert f'Environment="https_proxy={PROXY_URL}"' in content
    assert 'Environment="NO_PROXY=10.0.0.0/8,localhost"' in content
    assert restarted == [charm.SERVICE]


def test_removed_proxy_configs_are_cleared(drop_in: pathops.LocalPath, restarted: list[str]):
    # The drop-in file has proxy configs from an earlier event.
    drop_in.write_text(f'[Service]\nEnvironment="HTTPS_PROXY={PROXY_URL}"\n')
    ctx = testing.Context(charm.MyCharm)
    state_in = testing.State(model=testing.Model(proxy_config=testing.ProxyConfig()))

    ctx.run(ctx.on.update_status(), state_in)

    content = drop_in.read_text()
    assert PROXY_URL not in content
    assert 'Environment="HTTPS_PROXY="' in content
    assert restarted == [charm.SERVICE]


def test_unchanged_proxy_configs_dont_restart_the_workload(
    drop_in: pathops.LocalPath, restarted: list[str]
):
    ctx = testing.Context(charm.MyCharm)
    state_in = testing.State(
        model=testing.Model(proxy_config=testing.ProxyConfig(https=PROXY_URL)),
    )
    ctx.run(ctx.on.config_changed(), state_in)
    restarted.clear()

    ctx.run(ctx.on.update_status(), state_in)

    assert restarted == []
```

(manage-proxy-configs-integration-tests)=
### Write integration tests

See first: {ref}`write-integration-tests-for-a-charm`

In an integration test, change the proxy configs with [](jubilant.Juju.model_config), then check that the workload has the new values.

Juju doesn't fire an event when the proxy configs change, so the charm only reads the new values in its next event. To make the next event come sooner, set the `update-status-hook-interval` model config to a short interval for the test module. Then wait for the workload to have the new values, instead of waiting for a unit status, because the unit status doesn't change when the proxy configs change.

#### Test a Kubernetes charm

For a Kubernetes charm, read the Pebble plan in the workload container:

```python
import pathlib

import jubilant
import pytest
import yaml

APP = 'proxy-app'
UNIT = f'{APP}/0'
PROXY_URL = 'http://proxy.example.com:3128'


@pytest.fixture(scope='module', autouse=True)
def frequent_update_status(juju: jubilant.Juju):
    """Run update-status every 10 seconds, so the charm reads the new proxy configs sooner."""
    juju.model_config({'update-status-hook-interval': '10s'})
    yield
    juju.model_config(reset='update-status-hook-interval')


def workload_environment(juju: jubilant.Juju) -> dict[str, str]:
    """Return the environment of the workload service in the Pebble plan."""
    plan = juju.ssh(UNIT, '/charm/bin/pebble', 'plan', container='proxy-app')
    return yaml.safe_load(plan)['services']['proxy-app'].get('environment', {})


@pytest.mark.juju_setup
def test_deploy(charm: pathlib.Path, juju: jubilant.Juju):
    juju.deploy(charm, app=APP, resources={'proxy-app-image': 'ubuntu/nginx:latest'})
    juju.wait(jubilant.all_active, timeout=600)


def test_proxy_configs_reach_the_workload(juju: jubilant.Juju):
    juju.model_config({'juju-https-proxy': PROXY_URL})
    juju.wait(
        lambda _: workload_environment(juju).get('HTTPS_PROXY') == PROXY_URL,
        delay=5,
        timeout=120,
    )


def test_removed_proxy_configs_are_cleared(juju: jubilant.Juju):
    juju.model_config(reset='juju-https-proxy')
    juju.wait(
        lambda _: workload_environment(juju).get('HTTPS_PROXY') == '',
        delay=5,
        timeout=120,
    )
```

#### Test a machine charm

For a machine charm with a systemd service, read the environment of the running workload process. This also checks that the charm restarted the service after it changed the drop-in file:

```python
import pathlib

import jubilant
import pytest

APP = 'proxy-machine'
UNIT = f'{APP}/0'
SERVICE = 'my-workload.service'
PROXY_URL = 'http://proxy.example.com:3128'


@pytest.fixture(scope='module', autouse=True)
def frequent_update_status(juju: jubilant.Juju):
    """Run update-status every 10 seconds, so the charm reads the new proxy configs sooner."""
    juju.model_config({'update-status-hook-interval': '10s'})
    yield
    juju.model_config(reset='update-status-hook-interval')


def workload_environment(juju: jubilant.Juju) -> dict[str, str]:
    """Return the environment of the running workload process."""
    pid = juju.ssh(UNIT, 'systemctl', 'show', SERVICE, '--property=MainPID', '--value')
    environ = juju.ssh(UNIT, 'sudo', 'cat', f'/proc/{pid.strip()}/environ')
    return dict(entry.split('=', 1) for entry in environ.split('\0') if '=' in entry)


@pytest.mark.juju_setup
def test_deploy(charm: pathlib.Path, juju: jubilant.Juju):
    juju.deploy(charm, app=APP)
    juju.wait(jubilant.all_active, timeout=600)


def test_proxy_configs_reach_the_workload(juju: jubilant.Juju):
    juju.model_config({'juju-https-proxy': PROXY_URL})
    juju.wait(
        lambda _: workload_environment(juju).get('HTTPS_PROXY') == PROXY_URL,
        delay=5,
        timeout=120,
    )


def test_removed_proxy_configs_are_cleared(juju: jubilant.Juju):
    juju.model_config(reset='juju-https-proxy')
    juju.wait(
        lambda _: workload_environment(juju).get('HTTPS_PROXY') == '',
        delay=5,
        timeout=120,
    )
```

These examples use the `charm` fixture and the `juju` fixture from `pytest-jubilant`, as in {ref}`write-integration-tests-for-a-charm`.

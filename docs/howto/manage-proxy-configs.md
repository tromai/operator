(manage-proxy-configs)=
# How to manage proxy configs

Juju users can set the `juju-http-proxy`, `juju-https-proxy`, and `juju-no-proxy` {external+juju:ref}`model configs <list-of-model-configuration-keys>` to control how a charm and its workload make HTTP requests. A {ref}`mature charm <charm-maturity>` must respect these proxy configs.

## Read the proxy configs

Use [](ops.Model.proxy_config) to get the model's proxy configs. This returns an [](ops.ProxyConfig) instance:

```python
import ops


class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)

    def _get_proxy_environ(self) -> dict[str, str]:
        proxy = self.model.proxy_config
        no_proxy = ','.join(sorted(proxy.no))
        return {
            'HTTP_PROXY': proxy.http or '',
            'HTTPS_PROXY': proxy.https or '',
            'NO_PROXY': no_proxy,
            'http_proxy': proxy.http or '',
            'https_proxy': proxy.https or '',
            'no_proxy': no_proxy,
        }
```

[](ops.ProxyConfig) stores an empty `juju-http-proxy` or `juju-https-proxy` as `None`.

The `no` attribute of [](ops.ProxyConfig) contains the comma-separated entries of `juju-no-proxy` as a tuple of strings. An empty `juju-no-proxy` is an empty tuple. Entries in `no` can contain wildcards (`*.example.com`, or `*` for everything), domain suffixes (`.example.com`), or CIDRs (`10.0.0.0/8`).

Ops passes the proxy configs on as plain strings, and doesn't validate them. Juju doesn't fire an event when the proxy configs change, the charm can only observe the change the next time an event fires. Charms can read the proxy configs using the {ref}`reconciler pattern <handling-events-holistically>`:

```python
import ops


class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        self.container = self.unit.get_container('proxy-app')
        framework.observe(self.on['proxy-app'].pebble_ready, self._reconcile)
        framework.observe(self.on.config_changed, self._reconcile)
        framework.observe(self.on.update_status, self._reconcile)

    ...

    def _reconcile(self, event: ops.EventBase) -> None:
        if not self.container.can_connect():
            return
        proxy_env = self._get_proxy_environ()
        # Pass proxy_env to the workload, as described in the next section.
```

```{admonition} Best practice
:class: hint
:name: best-practice-proxy-url-handling

Don't include the proxy URLs in log messages or in the status message, as they can contain credentials. Tools accept the proxy configs in different formats, especially `no_proxy`. Validate the proxy configs before passing them to the workload.
```

## Manage proxy configs for the workload

The workload typically reads the proxy configs itself, so the charm needs to put the proxy configs where the workload expects them.

(manage-proxy-configs-kubernetes)=
### Kubernetes charms

When the workload runs in a sidecar container, the charm can pass the proxy configs with Pebble.

If the workload reads `*_proxy` or `*_PROXY` environment variables, create a Pebble layer that merges the variables into the existing service definition:

```python
class MyCharm(ops.CharmBase):
    ...

    def _proxy_layer(self, proxy_env: dict[str, str]) -> ops.pebble.Layer:
        return ops.pebble.Layer({
            'services': {
                'fastapi': {
                    'override': 'merge',
                    'environment': proxy_env,
                },
            },
        })
```

The examples on this page use the `proxy-app` container, which runs the [`api-demo-server`](https://github.com/canonical/api_demo_server) rock. The rock's Pebble layer defines a service called `fastapi`. The proxy layer only adds environment variables to that service, so add it after the layer that defines the service.

Because a [rock](https://documentation.ubuntu.com/rockcraft/stable/) defines its services, the reconciler can observe `pebble-ready` directly. `_get_proxy_environ` always returns every variable, so on the first event the comparison finds a difference and `replan()` starts the service with the proxy configs.

If the charm defines the service instead, in its `pebble-ready` handler, call the reconciler after adding the base layer. Then Pebble starts the service once, with the proxy configs, instead of starting it without them and restarting it:

```python
class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        self.container = self.unit.get_container('proxy-app')
        framework.observe(self.on['proxy-app'].pebble_ready, self._on_pebble_ready)
        framework.observe(self.on.config_changed, self._reconcile)
        framework.observe(self.on.update_status, self._reconcile)

    def _on_pebble_ready(self, event: ops.PebbleReadyEvent) -> None:
        layer: ops.pebble.LayerDict = {
            'services': {
                'fastapi': {
                    'override': 'replace',
                    'command': 'uvicorn api_demo_server.app:app --host=0.0.0.0 --port=8000',
                    'startup': 'enabled',
                }
            }
        }
        self.container.add_layer('base', layer, combine=True)
        self._reconcile(event)
        # Starts the service, if the reconciler didn't already start it.
        self.container.replan()

    ...
```

In the reconciler, add the layer and call [](ops.Container.replan). Pebble restarts the service only if the plan changed, so the charm doesn't restart the workload when the proxy configs are the same:

```python
class MyCharm(ops.CharmBase):
    ...

    def _reconcile(self, event: ops.EventBase) -> None:
        if not self.container.can_connect():
            return
        service = self.container.get_plan().services.get('fastapi')
        if service is None:
            # The proxy layer can only merge into a service that's already defined.
            return
        proxy_env = self._get_proxy_environ()
        self.container.add_layer('proxy-workload', self._proxy_layer(proxy_env), combine=True)
        self.container.replan()
```

[`cos-coordinated-workers`](https://github.com/canonical/cos-coordinated-workers/blob/main/src/coordinated_workers/worker.py) uses a similar approach: it rebuilds the Pebble layer in every event that it reconciles, and restarts the services if the layer differs from the current plan.

If the workload reads a configuration file, write the proxy configs to {ref}`a file in the workload container <files-in-containers-write-a-file>`.

```{note}
`replan()` only restarts a service if its configuration in the plan changed. That's enough for environment variables and CLI options, because they're part of the plan. A change to a configuration file isn't part of the plan, so `replan()` leaves the service running with the old file. In the reconciler, compare the rendered file with the file in the container, and only if they differ, push the new file and restart the service with [](ops.Container.restart).
```

A workload can read proxy configs from CLI options, which the charm passes on using a new Pebble layer. For example, [`trino-k8s-operator`](https://github.com/canonical/trino-k8s-operator/blob/main/src/charm.py) passes the proxy configs to its workload as JVM flags.

(manage-proxy-configs-machine)=
### Machine charms

In a machine charm, the charm and its workload run on the same machine, so the charm can write the proxy configs directly to where the workload reads them.

If the workload reads the proxy configs from a file, write the file with {external+charmlibs:ref}`pathops <charmlibs-pathops>`. There are two common cases:

- A systemd service that reads environment variables. Write a systemd drop-in file in `/etc/systemd/system/<service>.service.d/`. Then reload systemd and restart the service with {external+charmlibs:ref}`charmlibs-systemd <charmlibs-systemd>`. For example, the [`k8s`](https://github.com/canonical/k8s-operator/tree/main/charms/worker/k8s/src/charm.py) charm writes a drop-in file for the `containerd` snap service, and the [`vault`](https://github.com/canonical/vault-k8s-operator/tree/main/machine/src/charm.py) charm writes a drop-in file for the Vault snap service.
- A workload with its own configuration file. Render the proxy configs into the workload's configuration file, then restart or reload the workload.

You can also pass the proxy configs to the workload through its CLI options or API:
- A snap workload can be configured with {external+charmlibs:ref}`charmlibs-snap <charmlibs-snap>`.
- The workload's start-up command can have the proxy configs as CLI options.
- The [`lxd`](https://github.com/canonical/charm-lxd/tree/main/src/charm.py) charm uses the LXD API to set its `core.proxy_http`, `core.proxy_https`, and `core.proxy_ignore_hosts` options.

The reconciler is similar to a Kubernetes charm: convert [](ops.ProxyConfig) into the format that the workload expects, compare the result with what the workload has, and restart the workload only if they differ. For example, a systemd service:

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

    def _get_proxy_environ(self) -> dict[str, str]:
        ...  # The same as in "Read the proxy configs".

    def _reconcile(self, event: ops.EventBase) -> None:
        lines = ['[Service]']
        for name, value in self._get_proxy_environ().items():
            # systemd expands '%' specifiers in Environment=, and proxy URLs
            # can contain '%' in percent-encoded credentials.
            value = value.replace('%', '%%')
            lines.append(f'Environment="{name}={value}"')
        # ensure_contents only writes the file if the contents differ.
        if pathops.ensure_contents(PROXY_DROP_IN, '\n'.join(lines) + '\n'):
            systemd.daemon_reload()
            # A workload that isn't running reads the drop-in file when it starts.
            if systemd.service_running(SERVICE):
                systemd.service_restart(SERVICE)
```

Only restart the service if it's running. The reconciler runs in `config-changed`, which Juju emits before `start`, and the charm might also have stopped the workload on purpose, for example in an action that takes the unit out of service. Restarting would start the workload in both cases.

Systemd ignores an `Environment=` line that has an invalid `%` specifier, so without the `%` escape the workload silently loses the proxy config.

## Manage proxy configs for the charm process

The charm process can make its own HTTP requests, for example to download a file or call an API.

Refer to the libraries' documentation such as {external+python:mod}`urllib.request` and [requests | Advanced usage | Proxies](https://requests.readthedocs.io/en/latest/user/advanced/#proxies) for more information.

If the charm uses {external+python:func}`subprocess.run`, it can pass the proxy configs to the `env` argument.

## Test the feature

### Write unit tests

See first: {ref}`write-unit-tests-for-a-charm`

To set the proxy configs in a unit test, pass a [](ops.testing.ProxyConfig) to [](ops.testing.Model). A default `ProxyConfig()` creates a model with `juju-http-proxy` and `juju-https-proxy` as empty strings, and `juju-no-proxy` is set to `127.0.0.1,localhost,::1`.

We want to test that the proxy configs reach the workload, and in sync with the proxy model configs.

#### Test a Kubernetes charm

These tests are for `MyCharm` from {ref}`manage-proxy-configs-kubernetes`:

```python
import ops
from ops import testing

from charm import MyCharm

PROXY_URL = 'http://proxy.example.com:3128'


def workload_layer() -> ops.pebble.Layer:
    """Return the layer that defines the workload service, before any proxy configs."""
    return ops.pebble.Layer({
        'services': {
            'fastapi': {
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
    environment = plan.services['fastapi'].environment
    assert environment['HTTPS_PROXY'] == PROXY_URL
    assert environment['https_proxy'] == PROXY_URL
    assert environment['HTTP_PROXY'] == ''
    assert environment['NO_PROXY'] == '10.0.0.0/8,localhost'


def test_removed_proxy_configs_are_cleared():
    ctx = testing.Context(MyCharm)
    # The workload has proxy configs from an earlier event.
    proxy_layer = ops.pebble.Layer({
        'services': {
            'fastapi': {
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
    environment = plan.services['fastapi'].environment
    assert environment['HTTPS_PROXY'] == ''
    assert environment['https_proxy'] == ''
```

The reconciler runs in every `config-changed` and `update-status` event, so it accesses the container in your existing tests of those events too. Scenario raises an error if the container isn't in the state. Add `testing.Container('proxy-app')` to the state of those tests. The container can't connect to Pebble by default, so the reconciler does nothing.

#### Test a machine charm

For a machine charm, check the file or API that the charm writes the proxy configs to. Replace the parts of the charm that change the machine, so that the tests don't need root access or a real service.

The reconciler runs in every `config-changed` and `update-status` event, so it changes the machine in your existing tests of those events too. Define the fixtures in `conftest.py`, and apply them to every test in the module with `pytestmark = pytest.mark.usefixtures('drop_in', 'restarted')`.

These tests are for `MyCharm` from {ref}`manage-proxy-configs-machine`:

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
    monkeypatch.setattr(charm.systemd, 'service_running', lambda service: True)
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


def test_stopped_workload_is_not_restarted(
    monkeypatch: pytest.MonkeyPatch, drop_in: pathops.LocalPath, restarted: list[str]
):
    monkeypatch.setattr(charm.systemd, 'service_running', lambda service: False)
    ctx = testing.Context(charm.MyCharm)
    state_in = testing.State(
        model=testing.Model(proxy_config=testing.ProxyConfig(https=PROXY_URL)),
    )

    ctx.run(ctx.on.update_status(), state_in)

    # The workload reads the drop-in file when it starts.
    assert PROXY_URL in drop_in.read_text()
    assert restarted == []
```

(manage-proxy-configs-integration-tests)=
### Write integration tests

See first: {ref}`write-integration-tests-for-a-charm`

In an integration test, change the proxy configs with [](jubilant.Juju.model_config), then check that the workload has the new values.

Juju doesn't fire an event when the proxy configs change, so the charm only reads the new values in its next event. To make the next event come sooner, set the `update-status-hook-interval` model config to a short interval for the test module. Then wait for the workload to have the new values, instead of waiting for a unit status, because the unit status doesn't change when the proxy configs change.

#### Test a Kubernetes charm

For a Kubernetes charm, read the Pebble plan of the workload container. The test below deploys the charm with the `api-demo-server` rock, which is a small web server. The rock defines the `fastapi` service. The test only reads the Pebble plan, so it doesn't depend on what the workload does.

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
    # Run Pebble in the charm container, because a rock doesn't have the shell that
    # `juju ssh --container` needs.
    plan = juju.ssh(
        UNIT,
        'env',
        'PEBBLE_SOCKET=/charm/containers/proxy-app/pebble.socket',
        '/charm/bin/pebble',
        'plan',
    )
    return yaml.safe_load(plan)['services']['fastapi'].get('environment', {})


@pytest.mark.juju_setup
def test_deploy(charm: pathlib.Path, juju: jubilant.Juju):
    juju.deploy(
        charm,
        app=APP,
        resources={'proxy-app-image': 'ghcr.io/canonical/api_demo_server/api-demo-server:3.0.0'},
    )
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

The plan shows what the charm asks Pebble to run. To also check that Pebble restarted the service with the new values, read `/proc/<pid>/environ` of the service process, as in the machine example below. A rock like `api-demo-server` has no shell or `cat`, so this only works for an image that has them.

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

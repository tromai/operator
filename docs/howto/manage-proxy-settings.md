(manage-proxy-settings)=
# How to manage proxy settings

Juju users can set the `juju-http-proxy`, `juju-https-proxy`, and `juju-no-proxy` {external+juju:ref}`model configuration keys <list-of-model-configuration-keys>` to tell charms to send their HTTP traffic through a proxy. Juju doesn't apply these options to your workload. Your charm reads the values, then passes them on to the workload in the form that the workload expects.

Respecting these options is a criterion for a {ref}`mature charm <charm-maturity>`.

## Implement the feature

### Get the proxy settings

Use [](ops.Model.proxy_config) to get the model's proxy settings. This returns an [](ops.ProxyConfig) instance:

```python
class MyCharm(ops.CharmBase):
    def _on_pebble_ready(self, event: ops.PebbleReadyEvent):
        proxy = self.model.proxy_config
        environment = {}
        if proxy.https:
            environment['HTTPS_PROXY'] = proxy.https
        if proxy.no:
            environment['NO_PROXY'] = ','.join(proxy.no)
        ...
```

<!-- TODO make this into a paragraph instead. -->
Things to know about the values:

- `http` and `https` are `str | None`. Any empty string is read as `None`. The defaults of `juju-http-proxy` and `juju-https-proxy` are the empty string.
- `no` is a `tuple[str, ...]`. It contains the comma-separated entries of `juju-no-proxy`. An empty string is read as an empty tuple.
- Setting a model config doesn't fire any event. Therefore the charm can only read the latest value the next time an event fire.
- ops read those model configs as plain strings, and doesn't validate them. The URLs can be malformed, and entries in `no` can contain wildcards (`*.example.com`, or `*` for everything), domain suffixes (`.example.com`), or CIDRs (`10.0.0.0/8`).

### Pass the settings to the workload

Your charm needs to validate the values, and populate them in your workload, using values that your workload expects.

How you do this depends on the workload. For a Kubernetes charm, the usual approach is to set environment variables in the Pebble layer. For example:

```python
import urllib.parse

import ops

# The workload must always connect to these directly, in addition to the
# entries in the model's juju-no-proxy setting.
WORKLOAD_NO_PROXY = ('.svc', '.svc.cluster.local')


def validate_proxy_url(name: str, url: str) -> None:
    # Don't include the URL in the error. It might contain credentials.
    parsed = urllib.parse.urlparse(url)
    if parsed.scheme not in ('http', 'https') or not parsed.hostname:
        raise ValueError(f'invalid {name}: expected an http:// or https:// URL')


def proxy_environment(proxy: ops.ProxyConfig) -> dict[str, str]:
    """Return the environment variables for the workload service."""
    environment: dict[str, str] = {}
    for name, url in (('HTTP_PROXY', proxy.http), ('HTTPS_PROXY', proxy.https)):
        if not url:
            continue
        validate_proxy_url(name, url)
        # Tools disagree on the case of these variables, so set both.
        environment[name] = environment[name.lower()] = url
    if environment:
        # dict.fromkeys removes duplicates and keeps the order stable. A stable value
        # avoids needless workload restarts when the Pebble layer is replanned.
        no_proxy = ','.join(dict.fromkeys((*proxy.no, *WORKLOAD_NO_PROXY)))
        environment['NO_PROXY'] = environment['no_proxy'] = no_proxy
    return environment


class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework):
        super().__init__(framework)
        framework.observe(self.on['workload'].pebble_ready, self._update_workload)
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

`replan()` only restarts the service if the layer changed, so it's safe to call repeatedly.

For a machine charm, the equivalent is to write the values where your workload reads them. For example, a systemd drop-in file with `Environment=` lines, or the workload's own configuration file. Reload or restart the workload only if the content changed.

```{admonition} Best practice
:class: hint

Populate `no_proxy` according to your workload, not only from `juju-no-proxy`. Add the destinations that your workload must always reach directly. Set the proxy variables only when a proxy is configured, in the case that your workload reads.

Also check which `no_proxy` syntax your workload supports. Tools differ in how they treat CIDRs, leading dots, and wildcards. If the workload can't express an entry, translate it or drop it with a warning. For example, a Java workload takes a `|`-separated `-Dhttp.nonProxyHosts` list that can't contain CIDRs.
```

```{admonition} Best practice
:class: hint

Validate the values, and put the unit in blocked status when they're invalid. Don't include the proxy URL in log messages or in the status message, because it can contain credentials.
```

If your charm code makes its own HTTP requests, pass the proxy to your HTTP client explicitly, for example with the `proxies` argument of `requests`.

### Use a library for validation

The [`charms.proxylib`](https://github.com/canonical/charms.proxylib) library validates the values and renders them as a dictionary, a systemd drop-in, or Kubernetes environment variables. It also adds extra `no_proxy` entries for you:

```python
import charms.proxylib

env = charms.proxylib.environ(enabled=True, add_no_proxies=['.svc', '.svc.cluster.local'])
if env.error:
    self.unit.status = ops.BlockedStatus(str(env.error))
    return
```

Examples: [`cos-coordinated-workers`](https://github.com/canonical/cos-coordinated-workers) and [`mysql-operators`](https://github.com/canonical/mysql-operators) pass the settings to workloads through the Pebble layer. [`trino-k8s-operator`](https://github.com/canonical/trino-k8s-operator) validates strictly and converts the values to JVM options.

## Test the feature

You'll want to add unit and integration tests.

### Write unit tests

See first: {ref}`write-unit-tests-for-a-charm`

In your unit tests, use [](ops.testing.ProxyConfig) and the `proxy_config` attribute of [](ops.testing.Model) to set the model's proxy settings in the input `State`. Then run an event, and check what your charm did with the values.

For example, in `tests/unit/test_charm.py`:

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
    # The charm adds the in-cluster entries to the model's entries.
    assert env['NO_PROXY'] == '127.0.0.1,localhost,::1,10.0.0.0/8,.svc,.svc.cluster.local'
    assert state_out.unit_status == testing.ActiveStatus()


def test_no_proxy_configured():
    # Both None and the default ProxyConfig() mean that the user hasn't set a proxy.
    for proxy_config in (None, testing.ProxyConfig()):
        state_out = run(proxy_config)
        assert environment(state_out) == {}
        assert state_out.unit_status == testing.ActiveStatus()
```

`testing.ProxyConfig` has the same attributes as `ops.ProxyConfig`, and its defaults follow Juju's defaults: empty `http` and `https`, and `('127.0.0.1', 'localhost', '::1')` for `no`.

`Model.proxy_config` is `None` by default. In that case the charm sees `ops.ProxyConfig(http=None, https=None, no=())`.

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

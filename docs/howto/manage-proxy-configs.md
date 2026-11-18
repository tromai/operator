(manage-proxy-configs)=
# How to manage proxy configs

Juju users can set the `juju-http-proxy`, `juju-https-proxy`, and `juju-no-proxy` {external+juju:ref}`model configuration keys <list-of-model-configuration-keys>` to influence how charm or charm workload makes any HTTP request. A {ref}`mature charm <charm-maturity>` must respect these proxy configs.

## Reading `juju-*-proxy` model configs

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

[](ops.ProxyConfig)'s `no` attribute contains the comma-separated entries of `juju-no-proxy` as a string tuple. An empty `juju-no-proxy` is read as an empty tuple. Entries in `no` can contain wildcards (`*.example.com`, or `*` for everything), domain suffixes (`.example.com`), or CIDRs (`10.0.0.0/8`).

`ops` exposes those model configs as plain strings, and doesn't validate them. Juju doesn't fire an event when the proxy configs change. To capture it, the charm must read the proxy configs in every event that it is interested in, following the :ref:`reconciler pattern <handling-events-holistically>`.

```python
class MyCharm(ops.CharmBase):
    def __init__(self, framework: ops.Framework) -> None:
        super().__init__(framework)
        framework.observe(self.on.proxy_app.pebble_ready, self._reconciler)
        framework.observe(self.on.config_changed, self._reconciler)
        framework.observe(self.on.update_status, self._reconciler)
        self.container = self.unit.get_container('proxy-app')

    # This can be provided by ops too.
    def _proxy_env(self) -> dict[str, str]:
        proxy = self.model.proxy_config
        return {
            # Empty string to disable any existing proxy config in the container.
            "HTTP_PROXY": proxy.http or '',
            "HTTPS_PROXY": proxy.https or '',
            "http_proxy": proxy.http or '',
            "https_proxy": proxy.https or '',
            # Keep it sorted so that the
            # same NO_PROXY won't trigger a service restart.
            "NO_PROXY": ",".join(sorted(proxy.no)),
            "no_proxy": ",".join(sorted(proxy.no)),
        }

    def _reconciler(self, _: ops.EventBase) -> None:
        # Early checks
        workload_ready = self.workload.is_ready
        if not workload_ready:
            return

        proxy_env = self._proxy_env()
        # Update the workload if proxy config changes.
```

## Manage proxy options for the workload

The workload typically reads proxy configs on its own. The charm needs to populate the proxy configs to where the workload expects.

Most workloads read the `*_proxy` or `*_PROXY` environment variables. Some read a configuration file or start-up flags (for example, [`trino-k8s-operator`](https://github.com/canonical/trino-k8s-operator) uses JVM flags).

### Manage proxy options for the workload on Kubernetes

The charms can pass the proxy configs with Pebble, when the workload is run in a side-car container.

If the workload expects environmental variables, create a new Pebble layer that merge over existing definition.

```python
class MyCharm(ops.CharmBase):
    ...

    def _proxy_config_layer(self, proxy_env: dict[str, str]) -> ops.pebble.Layer:
        return ops.pebble.Layer(
            {
                "services": {
                    "proxy-app": {
                        "override": "merge",
                        "environment": proxy_env,
                    }
                },
            }
        )
```

Then in your reconciler function, restarts the workload with `Container.replan()` only when the proxy configs changed from the existing layer specification.

```python
class MyCharm(ops.CharmBase):
    ...
    def _reconciler(self, _: ops.EventBase) -> None:
        # Early checks
        ...

        proxy_env = self._proxy_env()
        container_env = self.container.get_plan().services["proxy-app"].environment

        # This can be a helper function provided by ops.
        changed = False
        for proxy_name, proxy_val in proxy_env:
            if env := container.get(proxy_name):
                changed = env == proxy_val
            else:
                changed = True
                break

        if changed:
            self.container.add_layer(
                "proxy-workload",
                self._proxy_config_layer(proxy_env),
                combine=True,
            )
            self.container.replan()
```

[`cos-coordinated-workers`](https://github.com/canonical/cos-coordinated-workers/blob/main/src/coordinated_workers/worker.py) uses a similar approach: it rebuilds the Pebble layer on every reconcilable event and restarts the services if the layer differs from the current plan.

If the workload expects a configuration file, write the configs to a file in the workload container with {ref}`files-in-containers`.

### Manage proxy options for the workload on a machine charm

You can set the proxy for your machine workload

In a machine charm, the charm and its workload live in the same place. Therefore the charm can modify the machine environment directly:
- If the workload expects environmental variables, the charms can set the environment variables itself.
- If the workload expects a configuration file, generate the configuration into a file with {external+charmlibs:ref}`pathops <charmlibs-pathops>`.

Machine charms needs to install and manage system packages.

### Manage proxy options for the charm process

The charm can make its own HTTP requests. Because the charm process is short-lived, the proxy configs must be set before the charm makes any request.
Libraries such as {external+python:func}`urllib.request.urlopen` and `requests` by default detect the proxy configs from `*_proxy` environmental variables.

You can pass the proxy configs to those library instead. For example, pass a {external+python:class}`urllib.request.ProxyHandler` to {external+python:func}`urllib.request.build_opener`.

```python
import urllib.request


class MyCharm(ops.CharmBase):
    def get_some_value_from_github_api(self):
        # `urllib` only reads `no_proxy` from the environment. We cannot
        # pass it here.
        custom_proxies = {
            'http': self.model.proxy_config.http else '',
            'https': self.model.proxy_config.https else '',
            'http': self.model.proxy_config.http else '',
            'https': self.model.proxy_config.https else '',
        }
        proxy_handler = urllib.request.ProxyHandler(custom_proxies)
        opener = urllib.request.build_opener(proxy_handler)

        with opener.open('https://api.github.com') as response:
            return response.read().decode('utf-8')
```

For `requests`, see [requests | Advanced usage | Proxies](https://requests.readthedocs.io/en/latest/user/advanced/#proxies).

## Best practices

- Don't include the proxy URL in log messages or in the status message, as it can contain credentials.
- Set both lowercase and uppercase proxy environment variables, as tools can depend on either of them.
- Each tool receives proxy config differently, even different format for no_proxy, so check how your workload reads them.

## Test the feature

### Write unit tests

See first: {ref}`write-unit-tests-for-a-charm`

Needs to tailor to previous examples:

An example for k8s charm:

```python
# Set proxy configs
# Initialize the charm with a layer
# Assert that the layer contains the proxy configs
# Remove proxy configs
# Assert that proxy configs are removed from the layer
```

An example for machine charm:

```python
# Set proxy configs
# Initialize the charm
# Assert that a systemd config file is created with the proxy config
# Remove proxy configs
# Assert that proxy configs are removed from the systemd config file.
```

(manage-proxy-settings-integration-tests)=
### Write integration tests

See first: {ref}`write-integration-tests-for-a-charm`

In an integration test, change the proxy settings with [`juju.model_config`](jubilant.Juju.model_config), and check that the workload has picked up the values.

Because Juju doesn't emit an event when the model configuration changes, the charm only notices the change when the next event runs.

To test if the workload receives the latest proxy configs. Some ideas:
- K8s: Come up with a minimal API server that serves the proxy config options (one liner?). In the integration tests -> run the workload and then use an action to get the proxy configs for assertion.
- Machine: Can use `juju.ssh(unit, 'systemctl', 'show', 'my-service', '--property=Environment')` ?

<!--This test is for a Kubernetes charm, and reads the Pebble plan. For a machine charm, replace `workload_plan` with a command that shows the environment of your service, such as `juju.ssh(unit, 'systemctl', 'show', 'my-service', '--property=Environment')`.-->

<!--```python
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

See also: [](jubilant.Juju.model_config)-->

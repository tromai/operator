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

Any HTTP request from the charm and its workload (if available) must respect the proxy configs. The charm code should read the values from [](ops.Model.proxy_config)
and decide what to do with it.

Because Juju doesn't fire an event when the proxy configs change, read them in a handler that runs on every event. The charm should compare the new value with what the workload already uses, and restart the workload if the value changed. This is the approach that [`cos-coordinated-workers`](https://github.com/canonical/cos-coordinated-workers) uses: it rebuilds the Pebble layer on every event and restarts the services if the layer differs from the current plan.

<!--Both the charm and the workload can use this helper to turn [](ops.ProxyConfig) into environment variables:

 Can provide this from ops.
```python
def to_proxy_env(proxy: ops.ProxyConfig) -> dict[str, str]:
    """Return the proxy environment variables for the charm or the workload.

    ``no`` are sorted.
    Both lower and upper env variables are set.
    """
    environment: dict[str, str] = {}
    for name, url in (('HTTP_PROXY', proxy.http), ('HTTPS_PROXY', proxy.https)):
        if not url:
            url = ""
        environment[name] = environment[name.lower()] = url
    if environment:
        no_proxy = ','.join(sorted(proxy.no))
        environment['NO_PROXY'] = environment['no_proxy'] = no_proxy
    return environment
```

Keep the output of the helper stable. The same input must always give the same dictionary, in the same order. Otherwise, the charm sees a change on every event and restarts the workload each time.-->

### Manage proxy options for the charm

The charm can make its own HTTP requests, for example to download a file. Because the charm process is short-lived, the proxy configs must be set before the charm makes any request.
Libraries such as {external+python:func}`urllib.request.urlopen` and `requests` by default detect the proxy configs from `*_proxy` environmental variables.

```python
import os
import urllib.request


class MyCharm(ops.CharmBase):
    def get_some_value_from_github_api(self):
        os.environ.update(to_env(self.model.proxy_config))
        with urllib.request.urlopen('https://api.github.com') as response:
            return response.read().decode('utf-8')
```

To avoid changing the environment, pass the values to the library instead. For example, pass a {external+python:class}`urllib.request.ProxyHandler` to {external+python:func}`urllib.request.build_opener`. `urllib` only reads `no_proxy` from the environment.

```python

import os
import urllib.request


class MyCharm(ops.CharmBase):
    def get_some_value_from_github_api(self):
        custom_proxies = {
            'http': self.model.proxy_config.http else '',
            'https': self.model.proxy_config.https else '',
        }
        proxy_handler = urllib.request.ProxyHandler(custom_proxies)
        opener = urllib.request.build_opener(proxy_handler)

        with opener.open('https://api.github.com') as response:
            return response.read().decode('utf-8')
```

For `requests`, see [requests | Advanced usage | Proxies](https://requests.readthedocs.io/en/latest/user/advanced/#proxies).

### Manage proxy options for the workload on Kubernetes

The workload can make its own HTTP requests, and it reads proxy configs on its own. The charm needs to populate the proxy configs to where the workload expects.

Most workloads read the `*_proxy` or `*_PROXY` environment variables. Some read a configuration file or start-up flags (for example, [`trino-k8s-operator`](https://github.com/canonical/trino-k8s-operator) uses JVM flags).

If the workload runs on another container, the charms needs to pass the proxy configs through Pebble.

If the workload expects environmental variables, set the proxy configs in the `environment` field of each service in the Pebble layer. Then `Container.replan()` restarts only the services whose configuration changed.

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

        environment = to_proxy_env(self.model.proxy_config)
        # If a *_proxy var is empty, does it override any value in the previous layers ?
        container.add_layer(
            'workload',
            {
                'services': {
                    'my-service': {
                        'override': 'merge',
                        'environment': environment,
                    }
                }
            },
            combine=True,
        )
        container.replan()
        self.unit.status = ops.ActiveStatus()
```

If the workload expects a configuration file, write the configs to a file in the workload container with {ref}`files-in-containers`.

### Manage proxy options for the workload on a machine charm

For a machine charm, the charm can set the proxy configs for the workload directly.

If the workload reads environment variables, restart the workload with the env variables set (if proxy configs change).
If the workload reads a configuration file, use charmlibs.pathops to write to that file.
If the workload is a systemd service, writes to a systemd config file, for example `/etc/systemd/system/my-service.d/web-proxy.conf` (taken from https://github.com/canonical/charms.proxylib/tree/main)

```
[Service]
# Autogenerated by juju_app='<my-app>' for service='<my-service>'
Environment="http_proxy=http://example.com:8080"
Environment="HTTP_PROXY=http://example.com:8080"
Environment="https_proxy=https://example.com:8080"
Environment="HTTPS_PROXY=https://example.com:8080"
Environment="no_proxy=localhost"
Environment="NO_PROXY=localhost"
```

If the workload is a snap, use `snap set` or the snap's own configuration.

**Need to expand this section with a proper example.**

Only reload the workload when the proxy config changed.

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

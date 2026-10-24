# OpenStack (Epoxy) – Controller Installation Notes

Manual, package-based installation on Ubuntu, following the official OpenStack install guide.
All passwords below are placeholders: **`<PASSWORD>`** – store real values in a vault / `.env` that is never committed.

> Conventions: `controller` resolves to the controller management IP (`/etc/hosts`). Each service follows the same pattern:
> **database → service user/role → service + endpoints → config file → db sync → restart → verify**.

## 1. Base services

| Component | Package | Purpose | Config |
|---|---|---|---|
| NTP | `chrony` | Time sync between nodes | – |
| SQL | `mariadb-server python3-pymysql` | Service databases | `/etc/mysql/mariadb.conf.d/99-openstack.cnf` |
| Message queue | `rabbitmq-server` | Inter-service messaging | `rabbitmqctl add_user openstack <PASSWORD>` + `set_permissions openstack ".*" ".*" ".*"` |
| Cache | `memcached python3-memcache` | Token caching | `/etc/memcached.conf` |
| Key-value | `etcd-server` | Distributed coordination | `/etc/default/etcd` |

## 2. Keystone (Identity)
1. Create DB `keystone` and grant privileges to `'keystone'@'localhost'` and `'%'`.
2. Configure `/etc/keystone/keystone.conf` (DB connection, Fernet token provider).
3. Populate and initialize:
```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
keystone-manage bootstrap --bootstrap-password <PASSWORD> \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
4. Configure Apache (`ServerName controller`), create `admin-openrc`, then projects (`service`, `myproject`), user and role.
5. Verify: `openstack token issue`.

**Debugging tip:** `tail -100 /var/log/apache2/keystone.log`

## 3. Glance (Image)
- DB, user `glance` (role `admin` on project `service`), service `image`, public/internal/admin endpoints on `:9292`.
- `glance-api.conf`: DB connection, `[keystone_authtoken]`, `[paste_deploy] flavor = keystone`, file backend (`/var/lib/glance/images/`), `[oslo_limit]` section.
- Grant `reader` role at system scope to the glance user, then `glance-manage db_sync`.
- Upload CirrOS and verify with `openstack image list`.

## 4. Placement
- DB, user `placement`, service `placement`, endpoints on `:8778`, `/etc/placement/placement.conf`.
- Verify: `placement-status upgrade check` and `openstack --os-placement-api-version 1.2 resource class list`.

## 5. Nova (Compute)
- Databases: `nova_api`, `nova`, `nova_cell0`.
- Controller packages: `nova-api nova-conductor nova-novncproxy nova-scheduler`.
- `/etc/nova/nova.conf`: `transport_url` (RabbitMQ), `[keystone_authtoken]`, `[service_user]`, `[placement]`.
- After compute nodes join: `nova-manage cell_v2 discover_hosts --verbose`; check with `openstack compute service list --service nova-compute`.

## 6. Neutron (Networking – Open vSwitch, self-service networks)
- DB, user, config files: `neutron.conf`, `plugins/ml2/ml2_conf.ini`, `openvswitch_agent.ini`, `l3_agent.ini`, `metadata_agent.ini`.
- Provider bridge:
```bash
ovs-vsctl add-br br-provider
ovs-vsctl add-port br-provider <provider-interface>   # e.g. ens33
```
- Connectivity test from the router namespace: `ip netns exec qrouter-<id> ping 8.8.8.8`

## 7. Horizon, Cinder, Heat
- Horizon dashboard at `http://controller/horizon`.
- Cinder: `openstack volume create --size 1 volume1` to validate block storage.
- Heat: `heat-api`, `heat-api-cfn`, `heat-engine` services (restart after config changes).

## 8. First instance
```bash
openstack server create --image "Ubuntu 22.04" --flavor m1.small \
  --network selfservice --key-name ubuntu-key ubuntu-instance
openstack floating ip create provider
openstack server add floating ip ubuntu-instance <FLOATING_IP>
ssh ubuntu@<FLOATING_IP>
```

## 9. Octavia (Load Balancer)
1. Create the `octavia` user in project `service` and an `octavia-openrc`.
2. Build the amphora image with **diskimage-builder** (`amphora-x64-haproxy.qcow2`) and upload it:
```bash
openstack image create amphora-x64-haproxy --disk-format qcow2 \
  --container-format bare --private --tag amphora --file amphora-x64-haproxy.qcow2
```
3. Create keypair, `lb-mgmt` network, security groups and the `o-bhm0` management port.
4. Fill `octavia.conf` with: service project ID, `lb-mgmt` network ID, security group ID, flavor ID, amphora image tag/owner.

## 10. Troubleshooting log (challenges & solutions)

| Problem | Cause | Solution |
|---|---|---|
| Controller loses external connectivity after Octavia setup | DHCP on `o-bhm0` (Octavia management port) injects a **default route** via the LB management network | Delete the route (`ip route del default via <lb-mgmt-gw> dev o-bhm0`) and add a dhclient hook `/etc/dhcp/dhclient-enter-hooks.d/octavia-no-default-route` that unsets `new_routers` for `o-bhm0` |
| Ping over `o-bhm0` / management network not working | Stale IP configuration on the interface | `ip route` to inspect, then `ip addr flush dev o-bhm0` and re-request DHCP |
| Octavia cannot boot amphorae | **Image owner ≠ project ID configured** (the amphora image was owned by a different project than `service`) | Compare `openstack image show amphora-x64-haproxy -c owner` with the configured project ID and align them |
| Keystone authentication errors | Misconfig of Apache/Keystone | Read `/var/log/apache2/keystone.log` |
| Glance API failing on startup | Missing `[oslo_limit]` credentials / system-scope reader role | Add `reader` role at system scope and fix `endpoint_id` |
| Heat API not responding | Services/memcached not running after config change | Enable and restart `memcached`, `apache2`, `heat-api`, `heat-api-cfn`, `heat-engine` |

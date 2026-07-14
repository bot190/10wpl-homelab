# Asterisk PBX External Configuration

This cluster deploys Asterisk for Cisco 7945 phones running SIP firmware.
Kubernetes owns the PBX and TFTP services, and `external-dns` publishes the
service DNS records from their LoadBalancer annotations.

## Kubernetes Addresses

The deployment reserves these MetalLB addresses:

| Service | DNS name | IP | Purpose |
| --- | --- | --- | --- |
| Asterisk | `pbx.${SECRET_DOMAIN}` | `192.168.16.127` | SIP signaling and RTP media |
| TFTP | `tftp.${SECRET_DOMAIN}` | `192.168.16.128` | Cisco phone boot/provisioning files |

`external-dns` should create the DNS records automatically. No manual DNS record
is expected unless external-dns is unhealthy or the Windows DNS zone is not
receiving RFC2136 updates.

## Windows DHCP Scope Options

Configure these options on the DHCP scope used by the phones:

| Option | Value | Notes |
| --- | --- | --- |
| `150` | `192.168.16.128` | Primary Cisco TFTP server option. Use the TFTP LoadBalancer IP. |
| `66` | `tftp.${SECRET_DOMAIN}` or `192.168.16.128` | Fallback TFTP server name/address. |
| `42` | Existing NTP server IP | Recommended so phone clocks and logs are correct. |
| Router/DNS | Existing network values | Keep normal gateway and DNS options for the phone VLAN/subnet. |

If the phones are on a dedicated voice VLAN, assign that VLAN through switch
CDP/LLDP-MED configuration. Do not rely on Asterisk for VLAN placement.

## Cisco 7945 Provisioning Files

Cisco 7945 SIP firmware normally requests a TFTP file named:

```text
SEP<MAC>.cnf.xml
```

The MAC address must be uppercase and contain no separators. For example,
phone MAC `00:11:22:33:44:55` requests:

```text
SEP001122334455.cnf.xml
```

The repo includes a starting template at:

```text
kubernetes/apps/default/asterisk/app/config/tftp/SEPAAAAAAAAAAAA.cnf.xml.template
```

For each phone:

1. Copy the template to `SEP<MAC>.cnf.xml` in the same directory.
2. Replace the line identity fields:
   - `phoneLabel`
   - `featureLabel`
   - `name`
   - `displayName`
   - `authName`
   - `authPassword`
   - `contact`
3. Set `authPassword` to the matching password from the encrypted
   `pjsip-auth.conf` entry in `asterisk-secret`.
4. Add the new file to `asterisk-tftp-configmap` in
   `kubernetes/apps/default/asterisk/app/kustomization.yaml`.
5. Commit and let Flux reconcile.

The initial Asterisk config defines extensions `100` through `109`, voicemail
access at `800`, and ring group `600`.

## Phone Boot Checklist

On first boot, verify the phone:

1. Receives an IP address from Windows DHCP.
2. Receives DHCP option `150` pointing at `192.168.16.128`.
3. Downloads `SEP<MAC>.cnf.xml` and `dialplan.xml` from TFTP.
4. Resolves `pbx.${SECRET_DOMAIN}` to `192.168.16.127`.
5. Registers to Asterisk as its assigned extension.

If a phone shows an `SCCP45...` load instead of `SIP45...`, it is not running
SIP firmware and will not register to this PJSIP deployment until it is
converted to a Cisco SIP firmware load.

## Validation Commands

After Flux applies the app, useful checks are:

```sh
kubectl -n default get pods,svc -l app.kubernetes.io/name=asterisk
kubectl -n default exec -it statefulset/asterisk -- asterisk -rx "pjsip show endpoints"
kubectl -n default exec -it statefulset/asterisk -- asterisk -rx "pjsip show contacts"
kubectl -n default logs deploy/asterisk-tftp
```

From a workstation on the phone network, test TFTP:

```sh
tftp 192.168.16.128 -c get dialplan.xml
```

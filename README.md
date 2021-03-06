# setup-dmvpn

This guide explains how to set up a Dynamic Multipoint VPN using `setup-dmvpn`.

## Setting Up the Certificate Authority

Install the Certificate Authority (CA) tool on a secure host:

<pre>apk add dmvpn-ca
</pre>

Configure the CA by editing `/etc/dmvpn-ca.conf`. In this example, the following
configuration is used:

<pre>hub:
  hosts:
  - hubs.example.com
  subnets:
  - '10.0.0.0/8'
  - '172.18.0.0/16'
  - 'fd00::/32'

crl:
  dist-point: 'http://crl.example.com/dmvpn-ca.crl'
  lifetime: 1800
  renewal: 1200
</pre>

The `hosts` attribute specifies the IPv4 addresses of the hubs or DNS name(s)
resolving to those. In this example, it is assumed that resolution of
`hubs.example.com` yields an A record for each hub.

The `subnets` attribute is a list of subnets used in the VPN. This should
include the address ranges of all sites and the GRE tunnel addresses. In this
example, the following IP address scheme is used:

The `crl` object should be left out unless the CRL distribution point will be
configured.

<table>
<tr><td></td><th>IPv4</td><th>IPv6</th></tr>
<tr><td>Hub GRE address</td><td>172.18.0.&lt;hub id&gt;</td><td>fd00::&lt;hub id&gt;</td></tr>
<tr><td>Site VPNc GRE address</td><td>172.18.&lt;site id&gt;.&lt;vpnc id&gt;</td><td>fd00::&lt;site id&gt;:&lt;vpnc id&gt;</td></tr>
<tr><td>Site subnet</td><td>10.&lt;site id&gt;.0.0/16</td><td>fd00:0:&lt;site id&gt;::/48</td></tr>
</table>

IPv6 addresses can be left undefined if only IPv4 is used in the VPN.

After setting up the CA configuration, generate the root key and certificate:

<pre>dmvpn-ca root-cert generate
</pre>

Create the configuration for hubs and sites. In this example, there are two
hubs and two sites. Each site has two VPN concentrators (VPNcs) for redundancy.

<pre>dmvpn-ca hub create
dmvpn-ca gre-addr add 172.18.0.1 hub 1
dmvpn-ca gre-addr add fd00::1 hub 1

dmvpn-ca hub create
dmvpn-ca gre-addr add 172.18.0.2 hub 2
dmvpn-ca gre-addr add fd00::2 hub 2

dmvpn-ca site add FIN
dmvpn-ca subnet add 10.1.0.0/16 site FIN
dmvpn-ca subnet add fd00:0:1::/48 site FIN
dmvpn-ca vpnc create site FIN
dmvpn-ca gre-addr add 172.18.1.1 site FIN vpnc 1
dmvpn-ca gre-addr add fd00::1:1 site FIN vpnc 1
dmvpn-ca vpnc create site FIN
dmvpn-ca gre-addr add 172.18.1.2 site FIN vpnc 2
dmvpn-ca gre-addr add fd00::1:2 site FIN vpnc 2

dmvpn-ca site add SWE
dmvpn-ca subnet add 10.2.0.0/16 site SWE
dmvpn-ca subnet add fd00:0:2::/48 site SWE
dmvpn-ca vpnc create site SWE
dmvpn-ca gre-addr add 172.18.2.1 site SWE vpnc 1
dmvpn-ca gre-addr add fd00::2:1 site SWE vpnc 1
dmvpn-ca vpnc create site SWE
dmvpn-ca gre-addr add 172.18.2.2 site SWE vpnc 2
dmvpn-ca gre-addr add fd00::2:2 site SWE vpnc 2
</pre>

Finally, generate the keys and certificates for the hubs and VPNcs:

<pre>dmvpn-ca cert generate
</pre>

This commands generates a PFX file for each hub and VPNc, for example:

<pre># ls
FIN_1.9D6JLGHlLmTG4bVR.pfx  SWE_1.caN3yapMTpZbIVP4.pfx  _1.hy62AqLIUJcFuT1U.pfx
FIN_2.fXbw4HwqkLXlIbtk.pfx  SWE_2.0BElySor2L8fm6e2.pfx  _2.cDLUvB8XALBkD2vP.pfx
</pre>

The encrypted file contains the individual certificate, the corresponding
private key, and the root certificate. The password is embedded in the file
name. The file should be renamed when using out-of-band delivery for the
password.

## <a name="crl"></a>Setting Up a CRL Distribution Point

In this example, the CA host serves also as the master CRL distribution point.
In addition, there may be other distribution points which periodically mirror
the CRL from the CA host. It is assumed that `ca.example.com` resolves to the
CA host and `crl.example.com` resolves to the IP addreses of all distribution
points.

Install the CRL distribution point package on the target host (CA host or
mirror):

<pre>apk add dmvpn-crl-dp
</pre>

If setting up a mirror, configure the master distribution point by creating a
file named `/etc/dmvpn-crl-dp.conf` with the following contents:
<pre>MASTER_CRL_URL=http://ca.example.com/dmvpn-ca.crl
</pre>

Activate CRL distribution by executing the following commands:

<pre>
dmvpn-crl-update
rc-update add lighttpd
rc-service lighttpd start
</pre>

## Setting Up a Hub

*Warning*: This procedure will automatically set up the `iptables` firewall
using `awall`. If you require any additional rules, such as allowing SSH access
to the host, you should configure those first. The easiest way to do so is to
use the `setup-firewall` utility:

<pre>apk add awall-policies
setup-firewall
</pre>

Install the `dmvpn` package on the host to be configured as a DMVPN hub. It is
assumed that the network configuration of the host is already in place.

<pre>apk add dmvpn
</pre>

Execute the setup tool using the hub's PFX file, answering the questions
prompted. The password is deduced from the file name unless renamed. Enter the
prefix lengths that uniquely identify the site. The default values are valid
for this example. The prefix length may vary among the sites, in which case the
maximum length should be given.

<pre>setup-dmvpn &lt;pfx file&gt;
</pre>

The hub is now operational and its firewall has been set up. Firewall for IPv6
(`ip6tables`) is set up by `setup-dmvpn` only if IPv6 addresses are defined for
the VPN. (`setup-firewall` sets it up if IPv6 is enabled in the kernel.)

## Setting Up a Site VPNc (Spoke)

Install the `dmvpn` package on the host to be configured as a DMVPN spoke. It
is assumed that the host is already configured as a router to the site subnet.

<pre>apk add dmvpn
</pre>

Execute the setup tool using the spoke's PFX file, answering the questions
prompted. The password is deduced from the file name unless renamed.

<pre>setup-dmvpn &lt;pfx file&gt;
</pre>

The spoke is now operational. Firewall rules are updated automatically if they
are managed using `awall`.

## Backing Up the CA

It may be a good idea to back up the configuration and the state of
the CA. This section describes one way to do so.

If you are using a firewall, allow outgoing SSH connections to the
backup host. If you set it up with `setup-firewall`, you can do this by
enabling the `adp-ssh-client` policy. This will allow SSH connections
to any host, though.

<pre>awall enable adp-ssh-client
awall activate
</pre>

Generate an SSH key pair on the CA host:

<pre>ssh-keygen
</pre>

Append the generated public key to the list of the authorized keys on
the backup host. Install `rsync` on the backup host:

<pre>apk add rsync
</pre>

Install `in-sync` on the CA host:

<pre>apk add in-sync
</pre>

Configure the backup host as the target in the CA host's
`/etc/in-sync.conf`:

<pre>TARGET_HOSTS="backup.ca.example.com"
</pre>

Start the synchronization service on the CA host:

<pre>rc-update add in-sync
rc-service in-sync start
</pre>

### Disaster Recovery

In case the original CA host is lost, you may convert the backup host
to a new CA host by installing the CA tool:

<pre>apk add dmvpn-ca
</pre>

If the CA host was serving as the master CRL distribution point, you
need to [set up that function](#crl) as well.

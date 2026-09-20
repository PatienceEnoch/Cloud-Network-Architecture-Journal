# Hybrid Site-to-Site VPN: Connecting a Routed Linux Lab to AWS

I am using my Ubuntu routing lab as the on-premises side of a small hybrid-cloud network.

The local network is 10.10.10.0/24. The AWS VPC is 10.20.0.0/16. An Ubuntu Server sits at the edge of the local lab and runs strongSwan for IPsec.

## Architecture

~~~text
Ubuntu Client
10.10.10.10/24
      |
Ubuntu Server
10.10.10.1/24
routing + nftables + strongSwan
      |
      | IPsec
      |
AWS Virtual Private Gateway
      |
AWS VPC
10.20.0.0/16
~~~

The useful part of this lab is not simply getting a green tunnel status. I want to be able to explain every routing decision on both sides.

## The AWS objects finally make sense as roles

**Virtual Private Gateway**

This is the AWS-side VPN endpoint attached to the VPC.

**Customer Gateway**

This is AWS's representation of the external endpoint on my side of the connection. It does not replace strongSwan. It tells AWS what remote VPN device it is connecting to.

**Site-to-Site VPN connection**

This ties the two gateway definitions together and produces the actual tunnel configuration.

**strongSwan**

This is the IPsec implementation running on the Ubuntu Server. AWS defines its side of the VPN, but strongSwan still has to be configured with matching tunnel parameters.

## Private address versus public endpoint

One detail that became much clearer during the build is that these are different identities:

~~~text
10.10.10.1
~~~

is the Ubuntu Server's address inside the lab.

The AWS Customer Gateway instead references the public-facing address used to reach the local VPN endpoint across the Internet.

That distinction is basic NAT behavior, but the VPN configuration makes the difference concrete.

## Static routing for the first version

I chose static routing for this version so I can focus on IPsec and packet flow before adding BGP.

AWS needs to know that:

~~~text
10.10.10.0/24
~~~

is reachable through the VPN.

The local side needs a valid path toward:

~~~text
10.20.0.0/16
~~~

through the IPsec policy/tunnel.

Later, BGP would make route exchange dynamic, but it would also add another control-plane variable. Static routing keeps the first failure domain smaller.

## Current state

At the current stopping point:

- the local Ubuntu router is working
- strongSwan is installed and running
- the AWS VPC exists
- the Virtual Private Gateway is attached
- the Customer Gateway exists
- the Site-to-Site VPN connection is being provisioned
- no IPsec Security Association is up yet

That last point is expected. A running IKE daemon is not the same thing as a configured or established VPN.

## What I want to prove next

Once AWS exposes the tunnel parameters, I want to verify the system layer by layer:

1. IKE negotiation succeeds.
2. An IPsec Security Association appears.
3. The AWS route table knows how to return traffic to 10.10.10.0/24.
4. The local router forwards traffic toward 10.20.0.0/16.
5. Security groups allow the intended test traffic.
6. Packet captures and counters agree with the expected path.
7. A controlled tunnel failure produces evidence I can explain.

The point is not just connectivity. The point is being able to trace why the connectivity works, and exactly where it fails when it does not.

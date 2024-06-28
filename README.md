# User-Based Firewall
The code in this repo consists of two daemons and two Apache httpd plug-ins that, together with specific Linux nftables firewall rules, make up the User-Based Firewall (UBF). The UBF can act on both on both tcp and udp connections, and would normally be configured (via iptables or other firewall utilities) to inspect connections on ports numbered 1024 and above (see iptables_rules.example). The ruleset implemented only permits a connection when the connecting and listening processes are running as the same user, or the connecting process is a member of the primary group (egid) of the listening process. When used on a system which implements the user private group scheme and disables protocols other than TCP and UDP, these changes effectively prevent users sharing data via the network unless they are both members of the same supplemental group.

While the UBF does not directly affect code using infiniband verbs or RDMA, many such applications use a TCP connection as a control channel to set up their infiniband queue pairs (QPs) and thus can be effectively controlled by the UBF. This does not prevent applications from using the connection manager (CM) directly to set up their QPs, and any application that does this would not be controlled by the UBF.

The UBF has only been thoroughly tested with IPv4 connections, and IPv6 loopback connections.

## ident2
The ident2 daemon is a follow-up to ident (https://www.rfc-editor.org/rfc/rfc1413). It implements the same general idea of ident, but with more details, wider applicability (both TCP and UDP as opposed to just TCP), and a security model around who to release of information to.

Questions must be sent to the local ident2d daemon via a unix domain socket bound to the filesystem. File permissions on this socket controls who may ask ident2 questions. Questions to ident2d take the form of protocol (TCP or UDP), IP address (IPv4 or IPv6) and port number. The local ident2d daemon will then forward the question to a peer ident2d daemon according to the IP address, if necessary. Valid peer IP ranges configured in ident2d.conf, and questions outside the configured ranges are immediately returned without an answer. This forwarding is done by UDP packet with both a low (privileged) source and destination port. As questions are asked by lossy UDP, there is no guarantee of a response. Questions that are not answered due to lost UDP packets, because the host does not exist, or because the host is offline are indistinguishable. No retries are attempted by ident2d.

Successful responses to queries may include the PID, UID, GID, and supplemental groups of the process with an open file handle to the socket that is bound to a protocol, address, and port matching the question. Partial responses as possible, such as only reporting the UID as represented by the inode owner field on the socket, and will be represented in the flags of the response. A limit of 350 supplemental groups is enforced, which roughly equates to a 1400 MTU packet.

### ident2_precache
ident2_precache is a module that can be included into HPC applications by setting the LD_PRELOAD environment variable to it's path. It captures all creations of sockets and notifies ident2 to pre-populate it's cache of which socket is opened by which process. This can provide significant performance benefits over the exhaustive search method of finding this association normally used by ident2 when the system has a large number of processes running and a significant number of connections being made, though the improvement is minimal except at most extreme scales.

## netid
The netidd daemon integrates with Linux nftables to make decisions on packets according to the UBF rules using the Netfilter Queue project (https://www.netfilter.org/projects/libnetfilter_queue/). It listens on a Queue Number for packets sent to it by nftables rules and queries ident2 for more information about both the source and destination of the packet. If the packet meets the UBF ruleset, or the associated user account is configured as an exempt listener or exempt connector, an accept verdict will be issued and the packet will pass through to the application. Standard firewall connection tracking ensures that netid only needs to evaluate the first packet of a connection.

If netid does not receive sufficient information from ident2 before a configured timeout, it will silently drop the packet. This silent drop is intentional and allows for retransmits from higher level protocols to reattempt the connection, such as automatic retransmits of the TCP SYN packet by the kernel. These retransmits will be considered separately as independent packets queued to netid. This process accounts for the lossy nature of ident2 questions.

If netid determines that a packet would violate the UBF rules, it issues an ICMP reply with code 13 (Admin Prohibited) to the source address to provide user feedback and terminate future retries.

If netid receives an answer, but the answer indicates that no matching socket was found, it issues an ICMP reply with code 3 (Port Unreachable) to provide user feedback and terminate future retries. This is the same ICMP message that would be generated by the kernel if no socket was listening at the target port number in the absence of a filewall rule, and is specifically chosen to integrate well with MPI applications that have codepaths that "backoff, wait, and retry" to handle different MPI ranks launching with slight delays compared to other ranks.

## urlfw
urlfw is an Apache httpd mod_rewrite "RewriteMap prg" program. It is meant to be setuid root and will read the forward parameters as "username/forwardname\n" from stdin. It will then attempt to read a file named "forwardname" from the configured directory and return the destination url if the system account matching the authenticated web user has execute rights on the file. The forwardname can be configured to be provided either by a wildcard subdomain without URL rewriting, or as a URL path component with URL rewriting. The directory for the forward files is configured in the source code, but can be (and is expected to be) a symlink to a more logical place on a shared filesystem.

Prior to returning the destination url, urlfw will query ident2 for the listening socket and check a variant of the UBF rules based on the owner and group of the forward file: if group write access is present on the file, the forward file group must match the primary group of the listening process; else the group or user must match. This check can be disabled if the forward file is owned by root by setting the sticky bit on the file.

When combined with the HPC File Permission Handler, users are limited in who they can grant access to based on their ability to set file permissions. The urlfw system has a significant throughput limitation as mod_rewrite serializes all access to RewriteMap programs across all httpd threads, so it is best used in moderation for special situations.

## mod_rewrite_fwfunc
mod_rewrite_fwfunc is an Apache httpd module that implements a system of forwarding web requests to hostnames and ports while enforcing access restrictions based on the UBF rules. As a httpd module, it is thread-safe and does not suffer the scaling limitations of urlfw. However, it does require the user, or scaffolding provided to the user, to know the exact hostname and port number they wish to connect to. The hostname and port can be configured to be provided either by a wildcard subdomain without URL rewriting, or as a URL path component with URL rewriting.

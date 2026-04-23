# Pre-requisite CoreDNS

  - Take me to [Lecture](https://kodekloud.com/topic/prerequisite-coredns/)

In this section, we will take a look at **CoreDNS**

- In the previous lecture, we saw why you need a DNS server, how it can help manage name resolution in large environments with many hostnames and Ips, and how      you  can configure your hosts to point to a DNS server. In this article, we will see how to configure a host as a DNS server.

- We are given a server dedicated as the DNS server and a set of IPs to configure as entries in the server. There are many DNS server solutions out there; in       this  lecture, we will focus on a particular one – CoreDNS.

- So, how do you get core DNS? CoreDNS binaries can be downloaded from their Github releases page or as a docker image. Let’s go the traditional route. Download    the binary using curl or wget. And extract it. You get the coredns executable
```
$ curl -LO https://github.com/coredns/coredns/releases/download/v1.12.4/coredns_1.12.4_linux_amd64.tgz
$ tar -zxf coredns_1.12.4_linux_amd64.tgz
```

- Run the executable to start a DNS server. It, by default, listens on port 53, which is the default port for a DNS server.

- Now, we haven’t specified the IP to hostname mappings. For that, you need to provide some configurations. There are multiple ways to do that. We will look at     one.

- First, we put all of the entries into the DNS servers /etc/hosts file. Then, we configure CoreDNS to use that file. CoreDNS loads its configuration from a file   named Corefile.

- Here is a simple configuration that instructs CoreDNS to fetch the IP to hostname mappings from the file /etc/hosts. When the DNS server is run, it now picks     the IPs and names from the /etc/hosts file on the server.
```
.:53 {
    # Use /etc/hosts to resolve hostname
    hosts /etc/hosts {
        reload 1m
        fallthrough
    }
 
    # Forward unmatched queries to the host's resolver
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    log
    errors
}
```
- CoreDNS also supports other ways of configuring DNS entries through plugins. We will look at the plugin that it uses for Kubernetes in a later section.

- Read more about CoreDNS here:

  - https://github.com/kubernetes/dns/blob/master/docs/specification.md

  - https://coredns.io/plugins/kubernetes/


## Installation of CoreDNS

```
$ wget https://github.com/coredns/coredns/releases/download/v1.7.0/coredns_1.7.0_linux_amd64.tgz
coredns_1.7.0_linux_amd64.tgz

```

## Extract tar file

```
$ tar -xzvf coredns_1.7.0_linux_amd64.tgz
coredns
```

## Run the executable file

- Run the executable file to start a DNS server. By default, it's listen on port 53, which is the default port for a DNS server.

```
$ ./coredns

```

## Configuring the hosts file

- Adding entries into the `/etc/hosts` file.
- CoreDNS will pick the ips and names from the `/etc/hosts` file on the server.

```
$ cat > /etc/hosts
192.168.1.10    web
192.168.1.11    db
192.168.1.15    web-1
192.168.1.16    db-1
192.168.1.21    web-2
192.168.1.22    db-2
```

## Adding into the Corefile

```
$ cat > Corefile
. {
	hosts   /etc/hosts
}

```

## Run the executable file

```
$ ./coredns

```


#### References Docs

- https://github.com/kubernetes/dns/blob/master/docs/specification.md
- https://coredns.io/plugins/kubernetes/
- https://github.com/coredns/coredns/releases

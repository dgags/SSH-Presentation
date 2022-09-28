<center><h1>SSH - The Secure Shell</h1></center>
<div style="text-align: right">Presentation by David Gagliardi</div>

## What is ssh?
SSH is a network protocol that gives users a secure way to access computers over any network.  

SSH also refers to the suite of utilities that implement the SSH protocol. It provides strong password authentication and public key authentication, as well as encrypted data communications between two computers connecting over an open network, such as the internet.  



## What can I do with ssh?
In it's simplest form ssh is a way to login to a remote server

```
$ ssh dgags@grafana.sys.bigco.net
```
- It can do local and remote port forwarding
    - Let's say I can't reach the web-server directly on my grafana server, but my util server can. I can forward a local port through an ssh tunnel on the util server to send tcp traffic to the grafana server:   
    - `$ ssh -L 8443:grafana.sys.bigco.net:443 dgags@util.sys.bigco.net`
    - Then i can just use a browser to connect to `https://localhost:8443`
    - For remote port forwarding let's say my grafana server cannot reach the YUM server, but my laptop can. I can create a tunnel between the grafana server and yumrepo.sys.bigco.net:
    - `$ ssh -R 8443:yumrepo.sys.bigco.net:443 dgags@grafana.sys.bigco.net` and with a bit of extra configuration of the yum client on the grafana server, I am up and running.
- Can be used to run commands remotely on servers
    - `$ ssh dgags@grafana.sys.bigco.net 'ps -eaf | grep -i grafana'`
- Use it to securely copy files from one place to another
    - `$ scp some-local-file dgags@grafana.sys.bigco.net:/destination/path/`
    - `$ scp dgags@grafana.sys.bigco.net:/full/path/to/remote-file.txt /local/destination/`
- It has a SOCKS proxy built in
    - If my grafana server does not have access to the internet to download files via http(s) but it can ssh to a server that does have that access, I can setup a local socks proxy:
    - `dgags@grafana:~$ ssh -D 1080 dgags@util.sys.bigco.net`
    - Then on the grafana server I can configure a utility like curl to use the socks proxy for all of it's web requests:
    - `dgags@grafana:~$ echo "--socks5-hostname localhost" >> .curlrc`
- You can change the default behavior of your ssh client with a custom `~/.ssh/config` file
- Leverage the `ssh-agent` and `ssh-add` utilities to cache your private key passphrase so you only have to type it in once in between reboots, no more typing your passphrase every time you ssh to a server!
- There is much more as well. 


## Special use case Certificate based auth
Certificate based auth has changed how we use ssh at BigCo dramatically. There are several things that it has radically changed how ssh is administered here:

1. You no longer need to distribute your public key to the servers you want to log into
2. You can change your public/private keypair at any time without any harm
3. Access to your servers is managed via an access control list, via principals, that is owned on a per team basis
4. Access is controlled via Enterprise Identity Management
5. You get MFA when you get your signed public ssh key from the Website

### How does Certificate based auth work? (50,000 ft view)
On your laptop you run the `ssh-keygen` command to create a public/private keypair

- You upload your public key eg: `id_rsa.pub` to Website
- In the the Website UI you go to the Team the the Website admins made for you and make a new Application and Principal

On the server you need to modify the ssh daemon to tell it where to find the Certificate Authority's public keys and where to also find the file that contains the principal.

Thankfully, most servers built within the past couple of years here have a properly configured ssh daemon, and the Certificate Authority's public keys file already installed. It is the sole responsibility of the server admin though to put the correct "authorized principals file" on the server.

Principals files belong in the `/etc/ssh/authorized_pricipals` directory. This is an example of the contents of one:
```
my-team.my-app.my-principal
```

And that's it. You can put multiple principals inside that file too so that you can give access to completely different Teams, Apps, Principals, or any combination thereof.

The second part to this of course is the *NAME* of the file. The name is critically important and has nothing to do with the contents whatsoever. Most of the servers we have here are linux based and are overwhelmingly Centos or Ubuntu based. In the case of a Centos server, there is always a `centos` user that is created as part of the OS install. When you create a file called: `/etc/ssh/authorized_pricipals/centos` with your full Principal name inside it, all the people you have configured to have access to that principal in the the Website UI will be able to use the `jump` command to login to that server using the following syntax:

```
$ jump centos@myserver.sys.bigco.net
```

The reason this works is because you've previously uploaded your public key to the Certificate based auth site, and if you are part of the "team.app.principal" configuration on the Website you will get back a modified copy of your ssh public key that has been signed by the Certificate Authority back end, with all of your "team.app.pricipal" assignments attached to it. When you connect to a server configured to work with Certificate based auth, your signed ssh public key is presented to the server and it's attached principals are inspected to see if there is a match to the user you're trying to connect as, `centos` in this example, and the full principal in the `/etc/ssh/authorized_pricipals/centos` file. If there's a match, and the rest of the signed public key metadata is valid, the ssh daemon let's you in.

Again this is the tip of the iceberg, there are further tricks and conveniences that you can configure in your `~/.ssh/config` file to make using Certificate based auth, and the `jump` command easier.

Also, you can use Certificate based auth with Ansible to administer all of your servers as well. I've barely scratched the surface of SSH. It truly is a Swiss army knife of server administration.

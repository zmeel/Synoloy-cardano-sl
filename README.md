**System: Docker container running Debian:latest on Synology DS415+ (8 Gb RAM)**

Building this on a Synology DS415+ is a very lengthy proccess. It will take a few hours. After the build is finished you can run the [cardano-sl](https://github.com/input-output-hk/cardano-sl) node 24/7 in this container. This is especially convienant as we enter the Cardano Reward Era also known as [Shelley](https://cardanoroadmap.com/)

Get the Debian image:
<pre>
docker pull debian:latest
</pre>

Setup Docker: check your LAN IP address. My Gateway is 192.168.178.1 so I define my subnet as --subnet 192.168.178.0/24. If your Gateway is for example 192.168.0.1 then define subnet as --subnet 192.168.0.0/24.
<pre>
docker network create -d macvlan --subnet 192.168.178.0/24 --ip-range 192.168.178.250/3 --gateway 192.168.178.1 -o parent=ovs_eth0 cardano_net
docker run -d --name=cardano-sl --network cardano_net debian:latest tail -f /dev/null
</pre>


This is a minimal install of Debian so we have to add lots of stuff:
<pre>
apt-get update
apt-get upgrade
apt-get install apt-utils
apt-get install net-tools
apt-get install openssh-server
apt-get install -y git curl build-essential libgtk2.0.0 libasound2 libnotify-bin libgconf-2-4 libnss3 libxss1
apt-get install sudo
apt-get install nano
</pre>

Configure and start the SSH server with nohup: we need to create a SSH tunnel later on in the process. Every time you restart this container you have to restart the SSH server.
<pre>
echo 'root:screencast' | chpasswd
sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
echo "export VISIBLE=now" >> /etc/profile
mkdir /run/sshd
nohup /usr/sbin/sshd -D >/dev/null 2>&1 &
</pre>

_Prebuild specific installations:_
<pre>
curl https://nixos.org/nix/install | sh
curl -sSL https://get.haskellstack.org/ | sh
</pre>

To employ the signed IOHK binary cache:
<pre>
mkdir -p /etc/nix
nano /etc/nix/nix.conf
</pre>

add to /etc/nix/nix.conf
<pre>
binary-caches             = https://cache.nixos.org https://hydra.iohk.io
binary-caches-public-keys = hydra.iohk.io:f/Ea+s+dFdN+3Y/G+FDgSq+a5NEWhJGzdjvKNGv0/EQ=
build-users-group =
</pre>

Getting the Cardano-sl code:
<pre>
$ git clone https://github.com/input-output-hk/cardano-sl.git
$ cd cardano-sl
$ git checkout master
</pre>

Building the node: you can build all but 'cardano-sl-wallet' is enough for the backend
<pre>
nix-build -A cardano-sl-static --cores 0 --max-jobs 2 --no-build-output --out-link cardano-sl-1.0
nix-build -A cardano-report-server-static --cores 0 --max-jobs 2 --no-build-output --out-link cardano-sl-1.0
nix-build -A cardano-sl-auxx --cores 0 --max-jobs 2 --no-build-output --out-link cardano-sl-1.0
nix-build -A cardano-sl-explorer-static --cores 0 --max-jobs 2 --no-build-output --out-link cardano-sl-1.0
nix-build -A cardano-sl-wallet --cores 0 --max-jobs 2 --no-build-output --out-link cardano-sl-1.0
nix-build -A cardano-sl-tools --cores 0 --max-jobs 2 --no-build-output --out-link cardano-sl-1.0
nix-build release.nix -A connect.mainnetWallet -o connect-to-mainnet
</pre>

Starting Cardano-sl node in Synology DS415+ Docker container:
<pre>
./connect-to-mainnnet
</pre>
or run it like a deamon:
<pre>nohup ./connect-to-mainnet >/dev/null 2>&1 &</pre>


I'm running [Daedalus](https://github.com/input-output-hk/daedalus) on MacBook Pro in VM with a Debian Desktop and create a SSH tunnel to the Debian server on Synology NAS:

terminal 1:
<pre>
ssh -L 8090:localhost:8090 user@servername
</pre>
terminal 2:
<pre>
npm run hot-server
</pre>
terminal 3:
<pre>
npm run hot-start
</pre>

If you succeed in building the node and would like to test the Daedalus wallet you can send me some ADA on this address. I'm happy to confirm its arrival :-)

<pre>DdzFFzCqrht9ZXsGFpW1EDApRjFYih8VfHnyUs7y7nDxMytE3h63KTRkoQSUUb2sV6pdVqqyV1LFAF436S3Qrw7TtmiVoP2KQid83n2r</pre>

## License
Cardano SL is released under the terms of the [MIT license](https://opensource.org/licenses/MIT). Please see [LICENSE](https://github.com/input-output-hk/cardano-sl/blob/master/LICENSE) for more information.


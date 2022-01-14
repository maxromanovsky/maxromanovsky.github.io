---
layout: post
title: "Running ElasticSearch on 32-bit Linux machine"
category: elasticsearch
author: Maksym Romanowski
tags: [elasticsearch, ELK, 32bit, i586]

---
I have an old piece of hardware with [Atom D2700](https://ark.intel.com/content/www/us/en/ark/products/59683/intel-atom-processor-d2700-1m-cache-2-13-ghz.html) CPU, which according to ARK is capable of running x64 OS. Vendor, however, never released a BIOS with x64 support, and I was unable to find it on an Internet.

Aside this sad fact, that small PC have decent specs, including 4 gigs of RAM, which makes it a good candidate for a single-node ElasticSearch cluster. I have to collect logs from my Kubernetes cluster somewhere, right?

Unfortunately, Elastic [dropped an official `i586` support](https://www.elastic.co/support/matrix) long time ago, which totally makes sense from commercial perspective.

Well, thanks to Debian and Java we still can run ElasticSearch on top of 32-bit Linux!

**This tutorial is written for ElasticSearch OSS v7.5.1, and might not work for newer versions, as source code [might change](https://github.com/elastic/elasticsearch/commits/master/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java)!**

<!--more-->

I've chosen official `deb` OSS distribution w/o JVM over `tar.gz` and commercial because:
- Deb has convenient integration with systemd and follows Debian conventions for files and directories.
- I don't plan on using any additional components from the commercial distribution, and don't want to unintentionally break any license for modifications required to run ES on 32-bit OS.
- Bundled JVM is 64-bit, thus it's of no use to me.

First things first, we need to prepare our OS:

{% raw %}
```shell script
# I know, it's bad practice
sudo -i
export ES_VERSION=7.5.1

apt install -y openjdk-11-jdk seccomp
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-${ES_VERSION}-no-jdk-amd64.deb
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-i386
dpkg --force-all -i elasticsearch-oss-${ES_VERSION}-no-jdk-amd64.deb
```
{% endraw %}

After this installation you'll get the similar message once you try using `apt` again:
```
[...]
You might want to run 'apt --fix-broken install' to correct these.
The following packages have unmet dependencies:
elasticsearch : Depends: libc6 but it is not installable
[...]
```

In order to fix it you [need to modify](https://unix.stackexchange.com/questions/404444/how-to-make-apt-ignore-unfulfilled-dependencies-of-installed-package) `/var/lib/dpkg/status`: search for `elasticsearch` and remove `depends: libc6`.

Now the next (and most important) trick: enable 32-bit support in ElasticSearch itself. Even though it's written in Java (which is cross-platform), it uses certain native bindings.

Minor change should be applied to a single class, [`SystemCallFilter`](https://github.com/elastic/elasticsearch/commits/v7.5.1/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java):

{% raw %}
```diff
diff --git a/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java b/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java
index 59f8bd5daf7..6a6b360726e 100644
--- a/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java
+++ b/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java
@@ -243,6 +243,7 @@ final class SystemCallFilter {
         Map<String,Arch> m = new HashMap<>();
         m.put("amd64", new Arch(0xC000003E, 0x3FFFFFFF, 57, 58, 59, 322, 317));
         m.put("aarch64",  new Arch(0xC00000B7, 0xFFFFFFFF, 1079, 1071, 221, 281, 277));
+        m.put("i386",  new Arch(0x40000003, 0xFFFFFFFF, 2, 190, 11, 358, 354));
         ARCHITECTURES = Collections.unmodifiableMap(m);
     }
```
{% endraw %}

I am doing it on my macOS, which compiles Java much faster. Save `systemcallfilter-i386.patch` and navigate shell to the same directory (replace `<elasticsearch-host>` with your ES hostname):

{% raw %}
```shell script
export ES_VERSION=7.5.1

# Prepare build environment
curl -s "https://get.sdkman.io" | bash
sdk list java
sdk install java 12.0.2-open
sdk use java 12.0.2-open

# Clone the repo, apply the patch
git clone https://github.com/elastic/elasticsearch.git
cd elasticsearch
git checkout v${ES_VERSION}
git apply ../systemcallfilter-i386.patch

# Download the whole internet to compile a single class
./gradlew clean compileJava

# Copy compiled class and replace it in the existing JAR
mkdir -p ../org/elasticsearch/bootstrap
cp server/build/classes/java/main/org/elasticsearch/bootstrap/SystemCallFilter.class ../org/elasticsearch/bootstrap
cd ..
scp <elasticsearch-host>:/usr/share/elasticsearch/lib/elasticsearch-${ES_VERSION}.jar .
cp elasticsearch-${ES_VERSION}.jar elasticsearch-${ES_VERSION}.jar.bak
# https://stackoverflow.com/questions/1667153/updating-class-file-in-jar
jar -uf elasticsearch-${ES_VERSION}.jar org/elasticsearch/bootstrap/SystemCallFilter.class

# Copy modified JAR back to the i586 host
scp elasticsearch-${ES_VERSION}.jar <elasticsearch-host>:/tmp
```
{% endraw %}

Now we can continue (in the same shell with sudo and `ES_VERSION`):
{% raw %}
```shell script
# Replacing the ElasticSearch jar with the patched one
mv /tmp/elasticsearch-${ES_VERSION}.jar /usr/share/elasticsearch/lib/

# Replacing stripped JNA bindings (w/o i586 support) with the full, original version
wget -O /usr/share/elasticsearch/lib/jna-4.5.1.jar https://github.com/java-native-access/jna/raw/4.5.1/dist/jna.jar

# Using system JVM
echo "JAVA_HOME=/usr/lib/jvm/java-11-openjdk-i386" >> /etc/default/elasticsearch

# Configuring ElasticSearch single node to act as production-ready cluster
echo "network.host: 0.0.0.0" >> /etc/elasticsearch/elasticsearch.yml
echo "discovery.type: single-node" >> /etc/elasticsearch/elasticsearch.yml
echo "-Des.enforce.bootstrap.checks=true" >> /etc/elasticsearch/jvm.options

# Using half of memory for JVM heap, per official recommendation. Another part would be used for file system cache
sed -Ei 's/^-Xms.*/-Xms2g/g' /etc/elasticsearch/jvm.options
sed -Ei 's/^-Xmx.*/-Xmx2g/g' /etc/elasticsearch/jvm.options

systemctl enable elasticsearch.service
systemctl start elasticsearch.service

cat /var/log/elasticsearch/elasticsearch.log

# Checking file descriptors: https://www.elastic.co/guide/en/elasticsearch/reference/current/file-descriptors.html
curl localhost:9200/_nodes/stats/process?filter_path=**.max_file_descriptors

# And finally:
curl localhost:9200
```
{% endraw %}

Bingo! It's up and running in no time.

Just remember, in order to upgrade to the new ES version you have to:
1. Download fresh `deb` file & install it.
1. Modify `/var/lib/dpkg/status` to get rid of unmet apt dependency
1. Check if [`SystemCallFilter`](https://github.com/elastic/elasticsearch/commits/v7.5.1/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java) in your particular release (`v7.5.1` in this case) has changed. [`master` branch is already different](https://github.com/elastic/elasticsearch/commits/master/server/src/main/java/org/elasticsearch/bootstrap/SystemCallFilter.java), so it's just a matter of time.
1. Re-compile `SystemCallFilter.java` if necessary (after applying patch)
1. Re-package `elasticsearch-*.jar` with the patched `SystemCallFilter.class`
1. Check if `/usr/share/elasticsearch/lib/jna-*.jar` has changed (maybe version was bumped), and replace it with the proper full version

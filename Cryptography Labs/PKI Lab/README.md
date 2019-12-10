# PKI Lab

## Lab Overview

公钥加密是当今安全通信的基础，但是当通信的一侧将其公钥发送到另一侧时，它会受到中间人的攻击。根本的问题是，没有简单的方法来验证公钥的所有权，即给定公钥及其声明的所有者信息，我们如何确保公钥确实由声明的所有者拥有？公钥基础结构（PKI）是解决此问题的实用方法。

该实验室的学习目标是让学生获得有关PKI的第一手经验。SEED实验室有一系列针对公钥加密的实验室，而这个实验室则针对PKI。通过完成本实验中的任务，学生应该能够更好地理解PKI的工作原理，如何使用PKI保护Web以及如何用PKI克服中间人攻击。而且，学生将能够了解公钥基础结构中信任的根源，如果根信任被破坏，将会出现什么问题。本实验涵盖以下主题：

- Public-key encryption 公钥加密
- Public-Key Infrastructure (PKI) 公钥基础设施
- Certificate Authority (CA) and root CA 证书颁发机构(CA)和根CA
- X.509 certificate and self-signed certificate X.509证书和自签名证书
- Apache, HTTP, and HTTPS  
- Man-in-the-middle attacks 中间人攻击

## Lab Tasks

### Task 1: Becoming a Certificate Authority (CA)

证书颁发机构（CA）是颁发数字证书的受信任实体。数字证书通过证书的指定主体证明公钥的所有权。许多商业CA被视为根CA。在撰写本文时，VeriSign是最大的CA。想要获得商业CA颁发的数字证书的用户需要支付这些CA。

在本实验中，我们需要创建数字证书，但是我们不会支付任何商业CA。我们将自己成为根CA，然后使用该CA为其他人（例如服务器）颁发证书。在此任务中，我们将自己设为根CA，并为此CA生成证书。与通常由另一个CA签名的其他证书不同，根CA的证书是自签名的。根CA的证书通常预先加载到大多数操作系统，Web浏览器和其他依赖PKI的软件中。根CA的证书是无条件信任的。

配置文件openssl.conf。为了使用OpenSSL创建证书，您必须具有配置文件。配置文件通常具有扩展名.cnf。它由三个OpenSSL命令使用：ca，req和x509。可以使用Google搜索找到openssl.conf的手册页。
您也可以从/usr/lib/ssl/openssl.cnf获得配置文件的副本。将此文件复制到当前目录后，您需要按照配置文件中的指定创建几个子目录（请查看[CA_default]部分）： 

```shell
dir = ./demoCA # Where everything is kept
certs = $dir/certs # Where the issued certs are kept
crl_dir = $dir/crl # Where the issued crl are kept
new_certs_dir = $dir/newcerts # default place for new certs.
database = $dir/index.txt # database index file.
serial = $dir/serial # The current serial number
```

对于index.txt文件，只需创建一个空文件。对于串行文件，在文件中输入一个以字符串格式（例如1000）的数字。设置配置文件openssl.cnf之后，就可以创建和颁发证书。

证书颁发机构（CA）。如前所述，我们需要为我们的CA生成一个自签名证书。这意味着该CA是完全受信任的，并且其证书将用作根证书。您可以运行以下命令为CA生成自签名证书：

```shell
$ openssl req -new -x509 -keyout ca.key -out ca.crt -config openssl.cnf
```

系统将提示您输入信息和密码。不要丢失此密码，因为每次要使用此CA为他人签名证书时，您都必须键入密码。还将要求您填写一些信息，例如“国家名称”，“通用名称”等。命令的输出存储在两个文件中：ca.key和ca.crt。文件ca.key包含CA的私钥，而ca.crt包含公钥证书。
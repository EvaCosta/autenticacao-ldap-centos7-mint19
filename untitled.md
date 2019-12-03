# Autenticando Usuários LDAP no Linux Mint 19.2

## 1. Instalando os pacotes

```text
$ sudo apt-get install libnss-ldap libpam-ldap slapd ldap-utils nscd
```

Caso prefira configurar a diretamente a partir dos arquivos basta apertar ESC até fecharem todas as janelas de configuração. ****

Caso opte pela configuração via interface a ordem está definida no tópico 2 do guia.

## 2. Altere o arquivo /etc/ldap.conf

1. Digite o endereço IP do servidor LDAP.
2. Insira o nome distinto da base de pesquisa. Valor deve corresponder aos valores no arquivo /etc/phpldapadmin/config.php dp servidor LDAP. **Importante: ldapi://** deve ser alterado para  **ldap://.**
3. A versão do LDAP deve ser a 3.
4. Insira os dados da conta administrativa do LDAP.
5. Adicione a senha.
6. O nome distinto a ser vinculado ao servidor.

\*\*\*\*

```yaml
$sudo vim /etc/ldap.conf
--------------------------------
                                                                                                                           
###DEBCONF###
##
## Configuration of this file will be managed by debconf as long as the
## first line of the file says '###DEBCONF###'
##
## You should use dpkg-reconfigure to configure this file via debconf
##

#
# @(#)$Id: ldap.conf,v 1.38 2006/05/15 08:13:31 lukeh Exp $
#
# This is the configuration file for the LDAP nameservice
# switch library and the LDAP PAM module.
#
# PADL Software
# http://www.padl.com
#

# Your LDAP server. Must be resolvable without using LDAP.
# Multiple hosts may be specified, each separated by a 
# space. How long nss_ldap takes to failover depends on
# whether your LDAP client library supports configurable
# network or connect timeouts (see bind_timelimit).

# The distinguished name of the search base.
base dc=user,dc=labredes,dc=info <--- (1)

# Another way to specify your LDAP server is to provide an
uri ldap://endereco-ip-do-ldapserver <--- (2)
# Unix Domain Sockets to connect to a local LDAP Server.
#uri ldap://127.0.0.1/
#uri ldaps://127.0.0.1/   
#uri ldapi://%2fvar%2frun%2fldapi_sock/
# Note: %2f encodes the '/' used as directory separator

# The LDAP version to use (defaults to 3
# if supported by client library)
ldap_version 3 <--- (3)

# The distinguished name to bind to the server with.
# Optional: default is to bind anonymously.
binddn cn=admin,dc=user,dc=labredes,dc=info <--- (4)

# The credentials to bind with. 
# Optional: default is no credential.
bindpw <senha> <--- (5)

# The distinguished name to bind to the server with
# if the effective user ID is root. Password is
# stored in /etc/ldap.secret (mode 600)
rootbinddn cn=admin,dc=user,dc=labredes,dc=info (6)

```

## 3. Altere o arquivo /etc/nsswitch.conf

Para cada parâmetro será feita uma consulta local antes da consulta na rede.

* files \(consultas locais\).
* ldap \(consultas realizadas no servidor remoto\).

Altere o arquivo ajustando o conforme mostrado abaixo acrescentando as informações que estiverem ausentes nas linhas indicadas:

```yaml
$ sudo vim /etc/nsswitch.conf
--------------------------------

# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         compat systemd ldap <--
group:          compat systemd ldap <--
shadow:         compat ldap <--
gshadow:        files ldap <--

hosts:          files mdns4_minimal [NOTFOUND=return] dns myhostname <--
networks:       files <--

protocols:      db files ldap <--
services:       db files ldap <--
ethers:         db files ldap <--
rpc:            db files ldap <--

netgroup:       nis ldap <--
```

## 4. Reinicie o serviço

```aspnet
# /etc/init.d/nscd restart
```

## 5. Configuração PAM

Durante a instalação do libnss-ldap ocorre a modificação do PAM, contudo é aconselhável verificar os arquivos.

* **/etc/pam.d/common-auth**
* **/etc/pam.d/common-account**
* /**etc/pam.d/common-password**

### 5.1 Arquivo **/etc/pam.d/common-auth**

```yaml
$ sudo vim /etc/pam.d/common-auth
--------------------------------

[...]
auth    [success=2 default=ignore]      pam_unix.so nullok_secure
auth    [success=1 default=ignore]      pam_ldap.so use_first_pass
# here's the fallback if no module succeeds
auth    requisite                       pam_deny.so
# this avoids us returning an error just because nothing sets a success code
# since the modules above will each just jump around
auth    required                        pam_permit.so
[...]
```

### 5.2  Arquivo **/etc/pam.d/common-account**

```yaml
$ sudo vim /etc/pam.d/common-account
--------------------------------

# here are the per-package modules (the "Primary" block)
account [success=2 new_authtok_reqd=done default=ignore]        pam_unix.so
account [success=1 default=ignore]      pam_ldap.so
# here's the fallback if no module succeeds
account requisite                       pam_deny.so
# prime the stack with a positive return value if there isn't one already;
# this avoids us returning an error just because nothing sets a success code
# since the modules above will each just jump around
account required                        pam_permit.so
```

### 5.3 Arquivo **/etc/pam.d/common-password**

```yaml
$sudo vim /etc/pam.d/common-password
--------------------------------

password        [success=2 default=ignore]      pam_unix.so obscure sha512
password        [success=1 user_unknown=ignore default=die]     pam_ldap.so use_authtok try_first_pass
# here's the fallback if no module succeeds
password        requisite                       pam_deny.so
# prime the stack with a positive return value if there isn't one already;
# this avoids us returning an error just because nothing sets a success code
# since the modules above will each just jump around
password        required                        pam_permit.so
```

### 5.4 Arquivo **/etc/pam.d/common-session**

Adicione a seguinte linha no arquivo:

```yaml
$ sudo /etc/pam.d/common-session
--------------------------------

session optional pam_mkhomedir.so skel=/etc/skel umask=0022
```

A linha acima criará um diretório HOME para usuários LDAP que não possuem diretório inicial ao efetuar login no servidor LDAP.

### 5.5 Arquivo **/etc/pam.d/common-session-noninteractive**

Altere o arquivo conforme as linhas abaixo:

```yaml
$sudo vim /etc/pam.d/common-session-noninteractive
--------------------------------

# here are the per-package modules (the "Primary" block)
session [default=1]                     pam_permit.so
# here's the fallback if no module succeeds
session requisite                       pam_deny.so
# prime the stack with a positive return value if there isn't one already;
# this avoids us returning an error just because nothing sets a success code
# since the modules above will each just jump around
session required                        pam_permit.so
# The pam_umask module will set the umask according to the system default in
# /etc/login.defs and user settings, solving the problem of different
# umask settings with different shells, display managers, remote sessions etc.
# See "man pam_umask".
session optional                        pam_umask.so
```

## 6 . Restarte o serviço para salvar as alterações:

```aspnet
# /etc/init.d/nscd restart
```

## 7. Testando a autenticação:

Nesse momento é possível obter os usuários da base ldap:

```aspnet
# getent passwd
--------------------------------

marlon:x:10001:10001:Marlon Silva:/home/marlon:/bin/bash
rafael:x:10002:10002:Rafael Alencar:/home/rafael:/bin/bash
diego:x:10003:10003:Diego Ferreira:/home/diego:/bin/bash
samuel:x:10004:10004:Samuel Martins:/home/samuel:/bin/bash
```

## 8. Configurando o gerenciador de exibição

É necessário alterar os arquivos de configuração do LightDM de forma que seja possível fornecer o nome e senha do usuário a ser logado na máquina, para isso é necessário substituir a sessão configurada pelo sistema.

Um arquivo de exemplo com toda as configurações possíveis está disponível em **/usr/share/doc/lightdm/lightdm.conf.gz**

Vamos criar um arquivo **/etc/lightdm/lightdm.conf.d /70-myconfig.conf.** 

Esse arquivo deverá ficar da seguinte forma:

```aspnet
[Seat:*]
user-session=mysession

#Ocultando a lista de usuários
greeter-hide-users=true

#Permitir login manual
greeter-show-manual-login=true
```

Reinicie o sistema do cliente e tente efetuar login com um usuário LDAP.

![Preencha o nome do usu&#xE1;rio para efetuar o login](.gitbook/assets/image%20%281%29.png)

![Preencha a senha do usu&#xE1;rio](.gitbook/assets/image%20%282%29.png)

Abra o terminal \(Ctrl + Alt + T \)e digite o comando pwd caso esteja tudo correto deve ser exibido o /home correspondente ao usuário.

![](.gitbook/assets/image%20%283%29.png)

## Referências

{% embed url="https://www.unixmen.com/configure-linux-clients-authenticate-using-openldap/" %}



{% embed url="https://www.vivaolinux.com.br/artigo/Cliente-Linux-no-servidor-LDAP?pagina=3" %}



{% embed url="https://wiki.ubuntu.com/LightDM" %}

{% embed url="https://stackoverflow.com/questions/14983807/ldap-login-works-via-terminal-but-doesnt-work-via-gui" %}


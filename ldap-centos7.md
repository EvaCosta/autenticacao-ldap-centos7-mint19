# Autenticando Usuários LDAP no CentOS 7

Curso Superior em Tecnologia em Sistemas para Internet.  
Instituto Federal de Educação Ciências e Tecnologia - Campus Barbacena.  
Disciplina: Administração e Gerência de Computadores  
Prof. Herlon Ayres Camargo

## Introdução

Esse guia tem por objetivo orientar a configuração do CentOS 7 e Linux Mint 19.2 para utilizar um servidor de diretório LDAP para autenticação.

É necessário ter um servidor de diretório LDAP instalado e funcionando.

## 1. Instalando os pacotes

Todas as etapas devem ser executadas pelo usuário 'root' ou com uso de 'sudo' no inicio de cada comando:

É necessária a instalação e configuração de dois módulos.

```text
# yum install nss-pam-ldapd nscd openssl
```

## 2. Configurando o arquivo /etc/nslcd.conf

O nss-pam-ldapd utiliza o mesmo arquivo para os módulos NSS e PAM. É necessário definir:

1. O URI que aponta para o servidor LDAP a ser usado para pesquisas de nome.
2. O nome da base de pesquisa.
3. O nome para vincular ao servidor.
4. As credenciais.
5. Especificar determinadas pesquisas de banco de dados.
6. Limite de tempo de conexão.
7. Limite de tempo de resposta.

**\# vim /etc/nslcd.conf**

```yaml
# This is the configuration file for the LDAP nameservice
# switch library's nslcd daemon. It configures the mapping
# between NSS names (see /etc/nsswitch.conf) and LDAP
# information in the directory.
# See the manual page nslcd.conf(5) for more information.

# The user and group nslcd should run as.
uid nslcd
gid ldap

# The uri pointing to the LDAP server to use for name lookups.
# Multiple entries may be specified. The address that is used
# here should be resolvable without using LDAP (obviously).
#uri ldap://127.0.0.1/
#uri ldaps://127.0.0.1/
#uri ldapi://%2fvar%2frun%2fldapi_sock/
# Note: %2f encodes the '/' used as directory separator
#uri ldap://127.0.0.1/
uri ldap://<ip-do-ldapserver> <-- (1)

# The LDAP version to use (defaults to 3
# if supported by client library)
#ldap_version 3

# The distinguished name of the search base.
#base dc=example,dc=com
base dc=user,dc=labredes,dc=info <-- (2)

# The distinguished name to bind to the server with.
# Optional: default is to bind anonymously.
binddn cn=admin,dc=user,dc=labredes,dc=info <-- (3)

# The credentials to bind with.
# Optional: default is no credentials.
# Note that if you set a bindpw you should check the permissions of this file.
bindpw <senha> <-- (4)

# The distinguished name to perform password modifications by root by.
#rootpwmoddn cn=admin,dc=example,dc=com

# The default search scope.
#scope sub
#scope one


# The distinguished name to perform password modifications by root by.
#rootpwmoddn cn=admin,dc=example,dc=com

# The default search scope.
#scope sub
#scope one
#scope base

# Customize certain database lookups.
base   group  ou=Grupos,dc=user,dc=labredes,dc=info <-- (5)
base   passwd ou=Usuarios,dc=user,dc=labredes, dc=info <-- (5)
#base   shadow ou=People,dc=example,dc=com
#scope  group  onelevel
#scope  hosts  sub

# Bind/connect timelimit.
bind_timelimit 30 <-- (6)

# Search timelimit.
timelimit 30 <-- (7)
```

## 3. Configure o nslcd

O nss-pam-ldapd utiliza uma daemon na procura de entradas do diretório, defina nslcd para iniciar automaticamente ao iniciar o sistema e ao reinicia-lo.

```text
# systemctl start nslcd
# systemctl enable nslcd
```

## 4. Crie um diretório inicial

```text
# mkdir /home/rafael
```

## 5. Resolvendo os UIDs e GIDs 

Altere o dono do  arquivo e as permissões de acesso.

```text
# chown 10002:10002 /home/rafael
# chmod 700 /home/rafael
```

## 6. Ativar o LDAP

Execute o comando para fazer as alterações necessárias para ativer o LDAP.

```text
# authconfig --updateall --enableldap --enableldapauth
```

## 7. Reinicie o nscd

```text
# systemctl restart nscd
# systemctl enable nscd
```

## 8. Ajustando o arquivo /etc/skel

É o diretório que contém os modelos de arquivo .bash\_profile e bashrc, esses arquivos são copiados para o diretório pessoal dos usuários \(/home\) ao serem criados, não sendo necessário realizar a configuração destes arquivos separadamente para cada usuário. 

Execute o comando para copiar esses arquivos para o /home do usuário criado.

```text
# cp /etc/skel/.bash* /home/rafael
```

## 9. Testando a autenticação

Deve retornar todos os usuários inclusive os pertencentes a base ldap:

```text
getent passwd
```

## Referências
<a href="https://www.server-world.info/en/note?os=CentOS_7&p=openldap&f=8">https://www.server-world.info/en/note?os=CentOS_7&p=openldap&f=8</a>

<a href="https://stato.blog.br/wordpress/ldap-parte-2-autenticando-usuarios-ldap-no-linux/">https://stato.blog.br/wordpress/ldap-parte-2-autenticando-usuarios-ldap-no-linux/</a>

<a href="https://www.lisenet.com/2016/setup-ldap-authentication-on-centos-7/">https://www.lisenet.com/2016/setup-ldap-authentication-on-centos-7/</a> 

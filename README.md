# sharelatex-rautu

Repositório para documentar a instalação, configuração e manutenção do
ShareLaTeX no servidor do LEG. 

## Instalando o ShareLaTeX no servidor

O ShareLaTeX pode ser instalado através do código fonte disponível no
GitHub, ou através do Docker (recomendado pelos desenvolvedores).

Para o servidor que é Ubuntu 14.04, foi necessário instalar o pacote
`docker` direto do site, pois a versão dos repositórios é muito antiga.
Veja em https://docs.docker.com/installation/ubuntulinux/ e instale com

```sh
sudo curl -sSL https://get.docker.com/ | sh
```

Para instalar o ShareLaTeX pelo docker, o site oficial é
https://github.com/sharelatex/sharelatex-docker-image. Mesmo seguindo os
passos ali dava "bad gateway", então segui o que esse pessoal fez em
https://github.com/sharelatex/sharelatex-docker-image/issues/2, e
instalei tudo com os seguintes comandos

```sh
sudo docker run -d --name sharemongo mongo:2.6.9
sudo docker run -d --name shareredis redis:latest
sudo docker run -d -P -p 5000:80 \
    -v ~/sharelatex_data:/var/lib/sharelatex \
    --env SHARELATEX_SITE_URL=http://localhost:5000 \
    --env SHARELATEX_MONGO_URL=mongodb://mongo/sharelatex \
    --env SHARELATEX_REDIS_HOST=redis --link sharemongo:mongo \
    --link shareredis:redis \
    --name sharelatex \
    sharelatex/sharelatex
```

A partir daqui ele já roda na porta 5000. Para criar o primeiro usuário
(admin):

```{r}
sudo docker exec sharelatex /bin/bash -c "cd /var/www/sharelatex/web; grunt create-admin-user --email user@email.com"
```

Para instalar uma vesão mais completa do LaTeX no container do
ShareLaTeX:

```sh
# sudo docker exec sharelatex tlmgr init-usertree # apenas se pedir
sudo docker exec sharelatex tlmgr update --self
sudo docker exec sharelatex tlmgr install scheme-full
```

## Configurando o Apache

As configurações do Apache estão em
https://github.com/sharelatex/sharelatex/wiki/Apache-as-a-Reverse-Proxy. Mas
usei um seção a mais que o Diego enviou, e ficou assim:

```
<VirtualHost *:80>
    ServerName sharelatex.leg.ufpr.br
    ProxyPreserveHost On

    RewriteEngine On
    RewriteCond %{REQUEST_URI}  ^/socket.io            [NC]
    RewriteCond %{QUERY_STRING} transport=websocket    [NC]
    RewriteRule /(.*)           ws://localhost:3026/$1 [P,L]

    <LocationMatch "/">
      ProxyPass http://localhost:5000/
      ProxyPassReverse http://localhost:5000/
    </LocationMatch>
</VirtualHost>
```


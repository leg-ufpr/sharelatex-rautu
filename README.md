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

### Configurando o nginx dentro do docker

Dentro do contâiner docker do sharelatex, o servidor utilizado é o
nginx. Por padrão, o sharelatex não permite fazer o upload de arquivos
com mais de 1 Mb, mas essa é uma restrição do servidor nginx. Para
resolver isso, foi necessário acessar o container do `sharelatex` com
`sudo docker exec -it sharelatex bash`. Depois, no arquivo de
configuração do nginx do sharelatex
(`/etc/nginx/sites-enabled/sharelatex.conf`) adicione a seguinte linha:

```
client_max_body_size 100M;
```

para permitir o upload de arquivos de até 100 Mb (ou qualquer outro
tamanho desejado).

## Integrando o knitr

Para integrar o knitr ao ShareLaTeX é necessário adicionar um script de
inicialização do `latexmk`. Segui os passos desse link
http://tex.stackexchange.com/questions/247369/install-knitr-on-own-sharelatex-server,
embora tive que fazer algumas modificações:

1. Entre no container do `sharelatex` com `sudo docker exec -it
   sharelatex bash`
2. Dentro do container, instale o R (adicionei o repositório do c3sl no
   `/etc/apt/sources.list`) e o latexmk `apt-get install r-base
   r-base-core r-base-dev latexmk`.
3. Instale o `knitr` com suas dependências.
4. Crie um arquivo `.latexmk` no HOME (nesse caso o HOME é `/root`) com
   o seguinte conteúdo:
   ```
	my $root_file = $ARGV[-1];
	
	add_cus_dep( 'Rtex', 'tex', 0, 'rtex_to_tex');
	sub rtex_to_tex {
		do_knitr("$_[0].Rtex");
	}
	
	sub do_knitr {
		my $dirname = dirname $_[0];
		my $basename = basename $_[0];
		system("Rscript -e \"library('knitr'); setwd('$dirname'); knit('$basename')\"");
	}
	
	my $rtex_file = $root_file =~ s/\.tex$/.Rtex/r;
	unless (-e $root_file) {
		if (-e $rtex_file) {
			do_knitr($rtex_file);
		}
	}
	```
	OBSERVAÇÃO: esses scripts de inicialiação do latexmk podem ficar no HOME
	com o `.latexmkrc`, ou em outros diretórios (veja no `man latexmkrc`)
	como: `/opt/local/share/latexmk`, `/usr/local/share/latexmk`,
	`/usr/local/lib/latexmk`, e o nome do arquivo deve ser `LatexMk` (com o
	mesmo conteúdo). O arquivo `LatexMk` também pode estar em
	`/etc/LatexMk`. No nosso caso, o script só funcionou depois de criar o
	`.latexmkrc` no HOME, e incorporar o conteúdo acima ao (já existente)
	`LatexMk` do `/etc`.
5. Saia do docker com `exit` e reinicie o sharelatex com `sudo docker restart sharelatex`.

## Configurando o email

TODO.

Para configurar o email, entrei no docker e inseri as informações em
`/etc/sharelatex/settings.coffee`, mas não funcionou. Também testei
colocar as informações em
`/var/www/sharelatex/web/config/settings.defaults.coffee`, mas também
não funcionou (mesmo depois de reiniciar o docker sharelatex).

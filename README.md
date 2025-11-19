# Manual de Configuração do vsftpd com Usuário `suporte`

Este manual documenta como configurar o servidor FTP **vsftpd** no Debian/Ubuntu, corrigindo o erro `530 Login incorrect` e permitindo que o usuário `suporte` acesse o diretório `/var/www/html`. Inclui também configuração de **modo passivo** e **FTPS (TLS)** para maior compatibilidade e segurança.

---

## 📦 Pré-requisitos

- Acesso root ou `sudo`.
- Pacotes necessários:
  ```bash
  sudo apt update
  sudo apt install vsftpd ftp curl openssl
👤 Passo 1: Criar e configurar o usuário

sudo adduser suporte
sudo passwd suporte
sudo usermod -d /var/www/html suporte
sudo usermod -s /bin/bash suporte

Verifique:
grep suporte /etc/passwd
# esperado: suporte:x:1001:1001::/var/www/html:/bin/bash

📂 Passo 2: Ajustar permissões


Se o usuário suporte será dono:
sudo chown -R suporte:suporte /var/www/html
sudo chmod -R 755 /var/www/html

Ou, se quiser manter Apache como dono mas permitir edição pelo grupo suporte:

bash
sudo chown -R www-data:suporte /var/www/html
sudo chmod -R 775 /var/www/html


⚙️ Passo 3: Configuração mínima do vsftpd
Editar /etc/vsftpd.conf:

conf
listen=YES
listen_ipv6=NO

local_enable=YES
write_enable=YES

chroot_local_user=YES
allow_writeable_chroot=YES

xferlog_enable=YES
log_ftp_protocol=YES
Reiniciar:

bash
sudo systemctl restart vsftpd
sudo systemctl status vsftpd

🔎 Passo 4: Testes

Teste local:

bash
ftp localhost
# Usuário: suporte
# Senha: definida
Teste com curl:

bash
curl -v --user suporte:SENHA ftp://localhost/
Teste remoto (FileZilla/WinSCP):

Host: IP do servidor (ex.: 10.100.43.151)

Porta: 21

Usuário: suporte

Senha: definida

🔥 Passo 5: Firewall e rede

Liberar portas:

bash
sudo ufw allow 21/tcp
sudo ufw allow 30000:30100/tcp   # portas passivas


🌐 Passo 6: Configuração de modo passivo
Adicionar ao /etc/vsftpd.conf:

conf
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=30100
pasv_address=SEU_IP_PUBLICO
pasv_address deve ser o IP público do servidor (ou IP da rede interna, se for apenas LAN).

Libere as portas 30000–30100 no firewall e roteador.

🔒 Passo 7: Configuração de FTPS (TLS)

Gerar certificado:

bash
sudo openssl req -x509 -nodes -days 365 \
  -subj "/CN=ftp.local" \
  -newkey rsa:2048 -keyout /etc/ssl/private/vsftpd.key \
  -out /etc/ssl/private/vsftpd.crt
sudo chmod 600 /etc/ssl/private/vsftpd.*
Configurar no /etc/vsftpd.conf:

conf
ssl_enable=YES
rsa_cert_file=/etc/ssl/private/vsftpd.crt
rsa_private_key_file=/etc/ssl/private/vsftpd.key

force_local_logins_ssl=YES
force_local_data_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
Reiniciar:

bash
sudo systemctl restart vsftpd
Conectar via cliente (FileZilla/WinSCP):

Tipo de conexão: FTP sobre TLS explícito

Porta: 21

Usuário: suporte

Senha: definida

🛡️ Segurança
Nunca usar root no FTP.

Bloquear SSH para suporte se quiser restringir apenas ao FTP:

bash
echo "DenyUsers suporte" | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart ssh


🚀 Troubleshooting
530 Login incorrect

Shell inválido → usermod -s /bin/bash suporte

Usuário listado em /etc/ftpusers → remover

Senha incorreta → passwd suporte

local_enable=YES faltando → ajustar no vsftpd.conf

Conexão recusada

Serviço não ativo → systemctl status vsftpd

Porta bloqueada → ufw allow 21/tcp

Erro ao iniciar vsftpd

Conflito de diretivas (listen=YES vs listen_ipv6=YES)

Certificado inválido em configuração TLS

Validar config: sudo vsftpd /etc/vsftpd.conf


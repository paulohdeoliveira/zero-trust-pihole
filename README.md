## Pi-hole & SSH via Tor v3 Autenticado

Este repositório documenta a arquitetura de segurança multicamadas (Defense in Depth) aplicada a um Raspberry Pi local, permitindo acesso administrativo (Painel Web e SSH) de qualquer lugar do mundo de forma anônima e segura.

##  Arquitetura de Segurança
* **Perímetro:** Firewall UFW bloqueando 100% das conexões de entrada externas.
* **Rede:** Serviço Oculto Tor v3 com autenticação obrigatória de cliente por Curva Elíptica (`x25519`).
* **Sessão:** Volatilidade de chaves no cliente (Descarte de credenciais após F5/reconexão).
* **Autenticação SSH:** Apenas chaves `ed25519` (Login root e senhas desativados) + Segundo Fator de Autenticação (2FA/MFA).

---

##  Passo a Passo da Implementação

### 1. Hardening do Sistema e Firewall (UFW)
A política padrão bloqueia qualquer entrada externa. O tráfego de saída é liberado para garantir a estabilidade das rotas do Tor.

```bash
# Definir políticas padrão
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir acessos apenas pela rede local interna
sudo ufw allow in from 192.168.0.0/24 to any port 22 proto tcp
sudo ufw allow in from 192.168.0.0/24 to any port 53 proto tcp
sudo ufw allow in from 192.168.0.0/24 to any port 53 proto udp
sudo ufw allow in from 192.168.0.0/24 to any port 443 proto tcp

# Ativar e validar
sudo ufw enable
sudo ufw status verbose
```

### 2. Geração das Chaves de Autenticação (x25519)
Script utilizado para gerar os pares de chaves estruturados estritamente em letras minúsculas (padrão obrigatório do Tor v3):

>[!NOTE]
> Direto no terminal
>
 ```
python3 -c "import base64; from cryptography.hazmat.primitives.asymmetric import x25519; private_key = x25519.X25519PrivateKey.generate(); public_key = private_key.public_key(); print('Cliente (Privada):', base64.b32encode(private_key.private_bytes_raw()).decode().replace('=', '').lower()); print('Servidor (Pública):', base64.b32encode(public_key.public_bytes_raw()).decode().replace('=', '').lower())"

```
>[!NOTE]
> Script generate_keys.py

```python
# generate_keys.py
import base64
from cryptography.hazmat.primitives.asymmetric import x25519

private_key = x25519.X25519PrivateKey.generate()
public_key = private_key.public_key()

priv = base64.b32encode(private_key.private_bytes_raw()).decode().replace('=', '').lower()
pub = base64.b32encode(public_key.public_bytes_raw()).decode().replace('=', '').lower()

print(f"Texto para o Servidor (.auth):\ndescriptor:x25519:{pub}\n")
print(f"Chave para o Cliente (Navegador/Torrc):\n{priv}")
```

### 3. Configuração do Serviço Oculto (Servidor)
No arquivo `/etc/tor/torrc`, configure o isolamento de processos e portas:

```text
HiddenServiceDir /var/lib/tor/pihole_secure_service/
HiddenServicePort 22 127.0.0.1:22
HiddenServicePort 443 127.0.0.1:443
```

#### Criação da pasta de credenciais e permissões rígidas:
```bash
sudo mkdir -p /var/lib/tor/pihole_secure_service/authorized_clients

# O conteúdo do arquivo .auth deve ser exatamente: descriptor:x25519:<CHAVE_PUBLICA_PLACEHOLDER>
sudo nano /var/lib/tor/pihole_secure_service/authorized_clients/cliente1.auth

# Aplicar permissões restritas do sistema
sudo chown -R debian-tor:debian-tor /var/lib/tor/pihole_secure_service/
sudo chmod 700 /var/lib/tor/pihole_secure_service/
sudo chmod 700 /var/lib/tor/pihole_secure_service/authorized_clients/
sudo chmod 600 /var/lib/tor/pihole_secure_service/authorized_clients/cliente1.auth

# Inicializar o serviço e capturar o endereço .onion gerado
sudo systemctl restart tor
sudo cat /var/lib/tor/pihole_secure_service/hostname
```

### 4. Configuração do Cliente (Tor Browser)
Para garantir que a chave não fique salva na máquina e suma após um **F5**, altere o parâmetro volátil no navegador:
1. Acesse `about:config` no Tor Browser.
2. Busque por `extensions.torlauncher.onionauth_persist`.
3. Altere o valor para **`false`**.

---
### Manutenção e Atualizações Automáticas
O servidor mitiga riscos de Zero-Days em softwares internos através do pacote `unattended-upgrades`, aplicando patches de segurança do Debian de forma automatizada.

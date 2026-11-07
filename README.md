# Lab Desafio: Amazon EC2 Instances

Laboratório desafio do curso AWS re/Start em que criei uma rede virtual (VPC), iniciei uma instância Amazon Linux no Amazon EC2, instalei um servidor web (Apache `httpd`) via *user data* e publiquei uma página HTML simples.

## Objetivos

- Configurar uma rede virtual (VPC, sub-rede, Internet Gateway e tabela de rotas).
- Colocar uma instância EC2 Amazon Linux nessa rede.
- Instalar um servidor web, implantar e executar uma aplicação simples nele.

## Arquitetura

```
Internet
   │
Internet Gateway
   │
VPC 10.0.0.0/16
   └── Sub-rede pública (1 AZ, IPv4 público automático)
          └── EC2 t3.micro (Amazon Linux 2023)
                 ├── Security Group: SSH (22) e HTTP (80)
                 └── Apache httpd em /var/www/html
```

## Serviços e recursos utilizados

| Recurso | Configuração |
|---------|--------------|
| Região | us-west-2 (Oregon) |
| VPC | Criada com o assistente "VPC e muito mais" (10.0.0.0/16) |
| Sub-rede | 1 sub-rede pública, IPv4 público de atribuição automática habilitado |
| Internet Gateway | Anexado à VPC, rota `0.0.0.0/0` na tabela de rotas |
| AMI | Amazon Linux 2023 |
| Tipo de instância | `t3.micro` (menor que medium, conforme o enunciado) |
| Volume raiz | gp2 (General Purpose SSD), 8 GiB |
| Security Group | SSH (22) e HTTP (80) com origem `0.0.0.0/0` |
| Acesso | EC2 Instance Connect (sem par de chaves) |

## Passo a passo

### 1. Criar a rede

1. Console da AWS → **VPC** → **Criar VPC** → **VPC e muito mais**.
2. Configurações:
   - CIDR IPv4: `10.0.0.0/16`
   - Zonas de disponibilidade: 1
   - Sub-redes públicas: 1 | Sub-redes privadas: 0
   - Gateways NAT: nenhum | Endpoints de VPC: nenhum
   - Habilitar nomes de host DNS e resolução de DNS
3. Em **VPC → Sub-redes**, selecionar a sub-rede pública → **Ações → Editar configurações da sub-rede** → habilitar **endereço IPv4 público de atribuição automática**.

O assistente já cria o Internet Gateway, a tabela de rotas com `0.0.0.0/0 → IGW` e a associação com a sub-rede.

### 2. Iniciar a instância EC2

**EC2 → Instâncias → Iniciar instâncias**:

- Nome: `web-server`
- AMI: Amazon Linux 2023
- Tipo: `t3.micro`
- Par de chaves: continuar sem par de chaves (acesso via EC2 Instance Connect)
- Rede: VPC e sub-rede criadas, **atribuir IP público automaticamente: habilitar**
- Security Group com regras de entrada para **SSH (22)** e **HTTP (80)**
- Armazenamento: 8 GiB, tipo **gp2**
- **Detalhes avançados → Dados do usuário:**

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable httpd
systemctl start httpd
chmod -R o+w /var/www/html
```

### 3. Validar a instalação do httpd

**Ações → Monitorar e solucionar problemas → Obter log do sistema.**

No log, a instalação aparece com as linhas `Installed: httpd-...`, `Complete!` e `Created symlink ... httpd.service`, e o cloud-init termina sem erros.

Trecho do log do sistema mostrando a instalação bem-sucedida:

```
cloud-init[1949]: Install  13 Packages
...
cloud-init[1949]:   Installing       : httpd-2.4.69-1.amzn2023.0.2.x86_64      13/13
...
cloud-init[1949]: Installed:
cloud-init[1949]:   httpd-2.4.69-1.amzn2023.0.2.x86_64
cloud-init[1949]:   httpd-core-2.4.69-1.amzn2023.0.2.x86_64
cloud-init[1949]: Complete!
cloud-init[1949]: Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service
cloud-init[1949]: Cloud-init v. 22.2.2 finished at Sat, 07 Nov 2026 15:29:13 +0000.
```

### 4. Conectar com EC2 Instance Connect

**Instâncias → selecionar a instância → Conectar → EC2 Instance Connect**, usuário `ec2-user`.

### 5. Criar e implantar a página

```bash
sudo tee /var/www/html/projects.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html>
<body>
<h1>Guilherme's re/Start Project Work</h1>
<p>EC2 Instance Challenge Lab</p>
</body>
</html>
EOF
```

Verificação:

```bash
sudo systemctl status httpd
ls -l /var/www/html
```

### 6. Testar no navegador

Acessar `http://<IPv4-público-da-instância>/projects.html`.

Resultado: a página foi retornada e exibida com sucesso.

![Página projects.html funcionando no navegador](screenshots/pagina-projects-html.png)

## Problema encontrado e solução

**Sintoma:** ao conectar pelo EC2 Instance Connect, o console mostrava *"Falha ao conectar à sua instância. Erro ao estabelecer conexão SSH com a sua instância."*

**Causa:** as regras de entrada do Security Group (SSH e HTTP) estavam restritas ao meu IP (`/32`). O EC2 Instance Connect no navegador conecta a partir de IPs da própria AWS, então a porta 22 era bloqueada.

**Solução:** editar as regras de entrada e trocar a origem para `0.0.0.0/0` nas portas 22 e 80. Depois disso a conexão funcionou normalmente.

> ⚠️ Liberar `0.0.0.0/0` é aceitável apenas em ambiente de laboratório. Em produção, o SSH deve ser restrito (por exemplo, a IPs conhecidos, ou usando Systems Manager Session Manager).

## Mensagens do log que não são erros

- `SSM Agent unable to acquire credentials`: a instância não tem perfil IAM do Systems Manager, e o lab não exige isso.
- `no authorized SSH keys fingerprints found for user ec2-user`: esperado, pois não foi usado par de chaves; o EC2 Instance Connect injeta uma chave temporária.
- `zram_generator ... too much memory`: aviso do kernel, pode ser ignorado.

## Aprendizados

- Montar uma VPC do zero: sub-rede pública, Internet Gateway e tabela de rotas.
- Usar *user data* para automatizar a configuração da instância na inicialização.
- Diferença entre a origem do tráfego do seu computador e a do EC2 Instance Connect ao configurar Security Groups.
- Ler o log do sistema (cloud-init) para validar a instalação de pacotes.

## Encerramento

Ao final, o lab foi finalizado com **End Lab**, o que encerra e remove os recursos criados.

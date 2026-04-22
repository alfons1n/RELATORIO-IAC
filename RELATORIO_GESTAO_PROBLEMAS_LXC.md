# Relatório de Gestão de Problemas: Provisionamento de Containers LXC (Clone vs OSTemplate)

## 1. Resumo do Incidente

Este relatório documenta as falhas encontradas ao tentar combinar parâmetros dinâmicos de inicialização gerenciados pelo Terraform (como senhas retiradas do Vault e inserção de chaves SSH) juntamente com a operação de clonagem de containers LXC no Proxmox. Após análise, as estratégias foram ajustadas, adotando o princípio **KISS (Keep It Simple, Stupid)**. O novo arranjo separa explicitamente as responsabilidades: a criação bruta e limpa fica a cargo do Terraform, enquanto as manipulações finas no sistema são puramente da alçada do Ansible/Semaphore.

## 2. Descrição Inicial do Problema

O primeiro problema foi identificado durante a criação de um template padrão para a clonagem de *containers* experimentais (para os alunos). O processo padrão exigia uma limpeza drástica no host de origem antes da conversão em template, minimizando assim a duplicação de registros confidenciais e evitando problemas de identificadores espelhados (como IDs de máquina e chaves Host do sistema).

Ao buscar contornar desafios com a inicialização automatizada de variáveis, elaborou-se um script bastante complexo em cima da instalação e adaptação do OpenSSH (para garantir acessibilidade na primeira reinicialização); os desafios mapeados incluíam:
- A impossibilidade de o serviço `ssh.service` subir por não possuir chaves de Host válidas após limpeza agressiva devido a problemas conhecidos de concorrência ou "condição de corrida" (*race conditions*).
- Tentativas malsucedidas de conectar via Semaphore após os clones ligarem, com falhas na execução normal devido aos bloqueios de chave e permissão, gerando rejeições por instabilidades do serviço isolado.

### 2.1 O Descompasso: Terraform OSTemplate vs LXC Clone

Descobriu-se ao consultar a documentação provedora que a API do Proxmox para instâncias LXC tem suportes distintos para as duas vias de lançamento:

- **Modo `ostemplate` (Adoção do Zero)**: Extremamente flexível para injetar chaves limpas iniciais (parâmetros `password` e `sshkeys`), desenhado para os servidores da infraestrutura raiz, em que a fundação e a premissa rodam logo após o deploy de arquivos .tar.gz locais ou por URL.
- **Modo `clone` (Réplica Base)**: Ao duplicar um container já configurado em ambiente real, **os parâmetros de inicialização limpa falham e não são suportados**, não permitindo reinserir a senha e as opções SSH dinamicamente via o código Terraform.

Os logs registraram perfeitamente as rejeições do uso via CLI e pipelines:
```text
Warning: Deprecated Resource
Error: 400 Parameter verification failed.
Error: Invalid index ... The given key does not identify an element in this collection value.
Error: Unsupported argument ... An argument named "sshkeys" is not expected here.
```

*Status de falha contínua do Terraform reportado pelo Semaphore:*
![Erro Semaphore](../imagens/relatorio/relatorio-01.png)
![Erro Trilha Proxmox](../imagens/relatorio/relatorio-details-proxmox-01.png)

No teste Ansible, a persistente resposta `UNREACHABLE` com `Connection refused` (já verificada via terminal Cockpit) confirmava o encerramento em *loop* do `sshd`.
![Falha Ansible Ping](../imagens/relatorio/relatorio-task-ansible-01.png)

## 3. Diagnóstico e Nova Estratégia Adotada

Dadas as características limitantes do Terraform e Proxmox, compreendeu-se que o verdadeiro "erro" não residia no código gerado pelo HCL do Terraform, e sim na sobrecarga indesejada transferida para os *scripts* internos e de limpeza para emular o papel do Ansible na primeira batida do boot.

Ao invés de tentar lidar com os engasgos contínuos do SSH, abraçamos e aplicamos as conclusões lógicas num modelo simplificado:

1. **Estabelecimento Mestre do Template Base** (`cockpit-install.sh` / `clear-lxc-template.sh`):
   - A dependência e segurança básica (Cockpit Integrado + Network Manager) já devem se fazer presentes.
   - A Chave SSH primária do Semaphore deve repousar tranquilamente na conta de administração autônoma em conjunto ao comando sudo de característica irrestrita (*NOPASSWD*). 
   - Apenas o `machine-id` deve ser meticulosamente expurgado. Com isso a "identidade de rede" via DHCP para a nova réplica está blindada contra anomalias (recebendo um IP verdadeiramente único).
2. **Orquestração Base do Terraform**: 
   - Exclusivamente acionará clones do template de origem. 
   - Modificará apenas metadados explícitos ao longo do Proxmox API (`hostname`, recursos, VMID e a vinculação VMBR).
3. **Múltipla Orientação Ansible (Semaphore)**:
   - Lançará as atualizações dinâmicas logo ao perceber a instância submissa comunicável (estado "UP"). Configura as permissões secundárias, novos alunos, perfis e *hardening* das senhas restritas.

## 4. O Desfecho Arquitetural Confirmado 🚀

Com as implementações enxugadas na base e retiradas as chaves arbitrárias de código do Vault nos módulos, não há mais restrição por *Race Conditions*: o SSH subia funcional por dispor das credenciais semânticas desde a clonagem.

O percurso transcorreu 100% de maneira profissional conforme atestado nas esteiras:
![Sucesso Terraform](../imagens/relatorio/relatorio-task-proxmox-sucess-01.png)
![Sucesso Ansible](../imagens/relatorio/relatorio-task-ansible-sucess-01.png)

### Resumo das Práticas:
- ❌ Alterações conturbadas da raiz no SSH por scripts de limpeza de Boot.
- ❌ Senhas efêmeras subindo para clones fechados sendo interceptadas.
- ❌ Provisões forçadas de SSH via Terraform contra as restrições Proxmox API.
- ✅ *Delegar a real injeção automatizada para os *Playbooks* do Ansible.*

O sucesso dos acessos do ping e a redução drástica de processamento em script refletem a passagem do estado prototípico amador aos caminhos de governança arquitetônica coesa no modelo profissional IAAC.

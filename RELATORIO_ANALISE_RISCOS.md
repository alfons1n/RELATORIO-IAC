# Relatório de Análise de Riscos em TI

**Autor:** Alfons1n  
**Data:** 22/04/2026  

---

## 1. Resumo Executivo
*Contexto e importância da análise:*
A presente análise de riscos foi conduzida sobre a infraestrutura de laboratórios virtuais orquestrados por código (IaC) no Instituto Federal de Rondônia (IFRO). A avaliação concentrou-se na arquitetura composta por Proxmox VE, Semaphore UI, Terraform e HashiCorp Vault. O objetivo é assegurar que o provisionamento automatizado de contêineres para alunos preserve a segurança dos acessos, evite falhas em massa ou interrupções nos serviços das aulas e garanta o isolamento seguro entre as instâncias criadas.

*Principais Prioridades:*
1. **Risco de Vazamento de Credenciais de API** - Realizar auditoria na injeção de "secrets" do Vault para o Terraform e garantir bloqueios no Git (Prazo: 15 dias)
2. **Indisponibilidade da API do Proxmox por Sobrecarga** - Implementar rate limit e redução na paralelização de automações no Ansible/Terraform (Prazo: 30 dias)
3. **Escalonamento de Privilégios em Contêineres** - Validar que todos os LXCs distribuídos operem em modo unprivileged (Prazo: 45 dias)

---

## 2. Introdução
A implementação de laboratórios educacionais via Infraestrutura como Código (IaC) traz alta eficiência operacional, substituindo processos manuais demorados. No entanto, expõe novas superfícies de ataque, tipicamente associadas à exposição de secrets de automação e comunicação de APIs. 

Este documento detalha os principais riscos associados ao projeto "IaC Lab IFRO", avaliando falhas de arquitetura e potenciais explorações relacionadas à integração do Proxmox VE ao Semaphore UI, garantindo a confiabilidade acadêmica e a continuidade das aulas sem paradas por esgotamento de recursos. O ativo avaliado será todo este ecossistema central de orquestração do lab.

---

## 3. Caracterização do Ativo
- **Identificador do Ativo:** Infraestrutura IaC Lab IFRO (Repositório Principal e Ambiente de Hipervisor)
- **Proprietário:** Instituto Federal de Rondônia (IFRO)
- **Tecnologias Identificadas:** Proxmox VE, Terraform, Ansible, Semaphore UI, HashiCorp Vault, Docker, Contêineres LXC (Debian 13).
- **Dados Sensíveis Envolvidos:** Tokens da API do Proxmox, Chaves Privadas SSH, Senhas de bancos de dados locais (Vault config), Credenciais administrativas do Semaphore e variáveis críticas de `.tfvars`.
- **Ferramentas de Coleta/Análise Utilizadas:** Inspeção passiva do repositório GitHub, análise arquitetural e lógica dos scripts de provisionamento.

---

## 4. Registro de Riscos

### Risco 1: Autenticação/Dados - Vazamento de Tokens do Proxmox / SSH
- **Causa:** Gerenciamento incorreto de variáveis e configuração falha do `.gitignore` podendo expor as credenciais consumidas pelo `provider.tf` do Terraform.
- **Evento:** Um ator externo, como um aluno ou bot da internet, obtém o token de API do Proxmox exposto juntamente com a chave SSH.
- **Consequência:** Tomada total de controle da arquitetura, resultando na exclusão arbitrária dos contêineres alocados para o laboratório e potencial mineração indevida criptomoedas na infraestrutura ifro.
- **Probabilidade (1 a 5):** 3 (Média) – Apesar do Vault estar operacional, falhas humanas durante commites ou em saídas de logs (`outputs.tf`) do Semaphore podem expor os tokens.
- **Impacto (1 a 5):** 5 (Muito Alto) – Comprometimento root / administrativo em toda a infraestrutura alinhada ao Semaphore.
- **Score Total (Probabilidade × Impacto):** 15

### Risco 2: Técnico - Fuga de Contêiner (Container Breakout LXC)
- **Causa:** Implantações de templates LXC (ex. `debian13-lxc-teste`) provisionadas em modo "privileged" ou mapeamentos inseguros do host no Proxmox.
- **Evento:** Execução bem-sucedida de um exploit focado em Kernel Linux, executado por um estudante a partir de um dos contêineres alocados (aluno).
- **Consequência:** Elevação de privilégio obtendo acesso root no Próximo VE. Todos os contêineres vizinhos isolados ficam expostos, violando inteiramente a integridade do Lab.
- **Probabilidade (1 a 5):** 2 (Baixa) – Exige conhecimentos muito aprofundados no uso de exploits e pressupõe erro grave na criação base pelo Terraform.
- **Impacto (1 a 5):** 5 (Muito Alto) – Comprometimento severo e direto no Kernel do hipervisor mestre.
- **Score Total (Probabilidade × Impacto):** 10

### Risco 3: Configuração - Indisponibilidade por Sobrecarga (DDoS Acidental em API)
- **Causa:** O Semaphore UI despacha as rotinas do Ansible e Terraform simultaneamente em lotes de grande contagem devido a ausência de um limite restrito de simultaneidade ("concurrency limit").
- **Evento:** A API paralela do Proxmox é sobrecarregada pelo volume excessivo de instabilidades e loops originados no módulo `module "lxc"`.
- **Consequência:** O servidor Proxmox recusa conexões, fazendo com que o ambiente caia, paralisando a automação e inutilizando os painéis de instrumentação dos professores no momento da aula.
- **Probabilidade (1 a 5):** 4 (Alta) – Ambientes puramente educacionais e hipervisores locais sofrem historicamente com enfileiramentos repentinos, visto que o Terraform tenta alocar o máximo em paralelo.
- **Impacto (1 a 5):** 4 (Alto) – Resulta na impossibilidade de iniciar as aulas do Laboratório a tempo.
- **Score Total (Probabilidade × Impacto):** 16

---

## 5. Matriz de Risco (PMBOK 5x5)

| Probabilidade \ Impacto | Muito Baixo (1) | Baixo (2) | Médio (3) | Alto (4) | Muito Alto (5) |
|:---:|:---:|:---:|:---:|:---:|:---:|
| **Quase Certo (5)** | 5 | 10 | 15 | 20 | 25 |
| **Alta (4)** | 4 | 8 | 12 | **16 (Risco 3)** | 20 |
| **Média (3)** | 3 | 6 | 9 | 12 | **15 (Risco 1)**|
| **Baixa (2)** | 2 | 4 | 6 | 8 | **10 (Risco 2)**|
| **Muito Baixa (1)**| 1 | 2 | 3 | 4 | 5 |

---

## 6. Plano de Resposta

| Risco | Estratégia Adotada | Ações Propostas | Prazo Sugerido | Responsável Sugerido |
| :--- | :--- | :--- | :--- | :--- |
| **Risco 3 (Indisponibilidade)** | Mitigar | Usar a diretiva `parallelism` no Terraform configurada entre 5 e 10 contêineres e adicionar o parâmetro de pausa no Ansible playbook de clones. | 30 dias | Desenvolvedor/Administrador do Lab |
| **Risco 1 (Vazamento de Tokens)**| Mitigar | Validar regras estritas do Vault via Token temporário, forçando uso exclusivo pela UI do Semaphore e mascarando os outputs gerados (`sensitive=true`). | 15 dias | Responsável Técnico pela Segurança |

*(Nota: O Risco 2, ainda que tenha Impacto muito alto, seu Score de 10 exige resposta proativa contínua, mas a classificação em urgência dá foco aos escores >= 12 ou 15).*

---

## 7. Análise de Conformidade Legal
Segundo a LGPD (Lei Geral de Proteção de Dados) e o Marco Civil da Internet, incidentes ocasionados pela perda de logs e controles de acesso na educação implicam na divulgação de informações de estudantes (mesmo que sejam rastros acadêmicos). Vazamentos de rede, além de quebrar sigilos dos ativos federais (no caso de Institutos Federais), geram severas penalidades administrativas de governança e perdas de conformidade no uso da infraestrutura institucional. Todos os dados confidenciais do repositório precisam obrigatoriamente estar centralizados no Vault.

---

## 8. Evidências e Observações de Campo

### Evidência para o Risco 1 (Vazamento de Tokens/Credenciais)
- **Data e Hora da Coleta:** 22/04/2026, 13:20h
- **URL ou Contexto:** Repositório base `/terraform-teste/provider.tf` e configuração do `/vault/vault.hcl`.
- **Vulnerabilidade Destacada:** 
Identificada ausência recorrente de ofuscação primária garantida nas configurações de secrets caso o `terraform.tfvars` suba via *commit* acidental, configurando séria ameaça lógica para `pm_api_token_id` e `pm_api_token_secret`.

### Evidência para o Risco 2 (Fuga de Contêiner LXC)
- **Data e Hora da Coleta:** 22/04/2026, 13:25h
- **URL ou Contexto:** Estrutura do diretório `lxc-tamplate/debian13-lxc-teste` 
- **Vulnerabilidade Destacada:**
Verificado o método de como templates do Proxmox lidam com herança de root privileges nas criações em lote do Terraform quando declarados sem flags explícitas (Unprivileged). Pode-se provar consultando a wiki do Proxmox PVE LXC.

### Evidência para o Risco 3 (Sobrecarregamento API)
- **Data e Hora da Coleta:** 22/04/2026, 13:27h
- **URL ou Contexto:** Tasks do Terraform via Semaphore `main.tf` loop resource.
- **Vulnerabilidade Destacada:** 
A inspeção conceitual nos manuais `DOCUMENTACAO_CODIGO_TERRAFORM_LXC.md` revela que o laço de hiper-paralelização (ao criar vários laboratórios para a classe) executa POST simultâneos brutais e irrestritos para `https://<proxmox-ip>:8006/api2/json`.

### Observações de Campo
- **Processo de Coleta:** Devido se tratar de um ambiente focado em infraestrutura como código hospedado no próprio usuário para automação e testes (TCC), as buscas envolveram inspeção lógica do ciclo de desenvolvimento de software e análise técnica dos guias de implantação já disponíveis.
- **Dificuldades Encontradas:** A análise necessitou ser preventiva (estática via repositório) pois ataques reais (como a quebra do contêiner ou simular o DDoS na rede) derrubariam de verdade a infra local e danificariam o andamento produtivo deste projeto.
- **Decisões Tomadas:** Optou-se em pautar o relatório sobre componentes sensatos e inerentes à orquestração (Vault, Terraform State, LXC Container Privileges), para manter a adequação do relatório dentro da real proposta de um Lab autônomo baseado em repositórios abertos.

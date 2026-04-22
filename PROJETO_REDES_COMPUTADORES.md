# CICLO DE VIDA DO PROJETO DE REDES DE COMPUTADORES

*Contexto do Projeto: Laboratório de Práticas de IaC (Infraestrutura como Código) – IFRO*

---

## 1. Fase Inicial – Autorização do Projeto

**Qual é o problema?**
A ausência de uma rede local (LAN) isolada e com capacidade adequada de tráfego para comportar o laboratório prático de provisionamento automatizado de contêineres. O uso da rede administrativa padrão gera concorrência de banda, afetando atividades administrativas e expondo o núcleo institucional aos testes práticos dos alunos.

**Qual o objetivo do projeto?**
Projetar e construir uma infraestrutura de rede robusta e segregada (lógica e fisicamente) para conectar o servidor "Proxmox VE" às estações de trabalho de 30 estudantes, provendo roteamento controlado, segurança e monitoramento do tráfego do laboratório.

**Por que esse projeto é importante?**
A estruturação da rede garante que as rotinas simultâneas do Terraform (via Semaphore UI) ocorram sem causar DoS (negação de serviço) em outros setores do campus, resguarda os equipamentos contra acessos não autorizados e promove a estabilidade e o isolamento dos laboratórios para fins educacionais.

---

## 2. Planejamento do Projeto: Definição dos objetivos, plano de ação

- **Número de usuários**: 30 alunos (simultâneos) e 1 Professor/Administrador.
- **Quantidade de computadores**: 30 Estações de Trabalho nas bancadas + 1 Servidor Físico de Virtualização (Proxmox / Docker).
- **Acesso à internet**: Link via fibra gerido de modo segregado com garantia de banda apenas para download de pacotes, dependências do Git e imagens do Docker/LXC.
- **Roteadores**: 1 x Firewall/Roteador (ex: Appliance pfSense ou Mikrotik) encarregado pelo DHCP da sub-rede do laboratório, NAT e controle de portas.
- **Switch**: 2 x Switches Layer 2/3 Gerenciáveis de 24 portas (Gigabit) com suporte a VLANs.
- **Lista de Equipamentos**:
  - 30 Máquinas Desktop
  - 1 Servidor de Rack (Hipervisor Proxmox VE)
  - 2 Switches 24P Giga
  - 1 Roteador Borda
  - Conjunto de Cabeamento Cat6 (Bobinas, Patch Panels, Patch Cords e conduletes).
- **Cronograma**:
  - *Semana 1*: Passagem da infraestrutura física (eletrocalhas e cabeamento CAT6).
  - *Semana 2*: Instalação dos equipamentos no Rack, conectorização e configuração das VLANs nos switches.
  - *Semana 3*: Configuração do DHCP, restrições e políticas de acesso (Roteador). Configuração IP das estações dos alunos.
  - *Semana 4*: Homologação em massa e testes de carga da rede usando Ansible localmente (+ entrega).

---

## 3. Execução do Projeto: Coordenação para realização do projeto

**Como a rede será montada?**
O cabeamento seguirá as normas de cabeamento estruturado (TIA/EIA-568) partindo das estações de bancada, distribuídas por eletrocalhas suspensas, até o rack centralizado na sala, onde ficam fixados os patch panels, os dois switches de 24 portas e o servidor Proxmox.

**Como os equipamentos serão conectados?**
As estações de mesa (1-30) serão conectadas simetricamente aos Switches (nas portas de acesso configuradas numa "VLAN ALUNOS"). O servidor Proxmox será conectado numa porta definida como *Trunk/Tagged* para permitir o ingresso tanto do tráfego nativo gerencial quanto o tráfego isolado de cada contêiner dos estudantes. Uma porta *Uplink* unirá os switches ao roteador.

**Como os IPs serão configurados?**
- **Infraestrutura/Servidores:** IPs Estáticos de Gerência. Exemplo: Roteador (`192.168.10.1`), Proxmox (`192.168.10.10`) e Semaphore (`192.168.10.20`).
- **Alunos:** Range dinâmico ofertado por um servidor DHCP alocado no roteador. Exemplo: Rede `192.168.20.0/24` do range `.100` até `.150`.

---

## 4. Monitoramento e Controle: Para garantir que os objetivos sejam alcançados

**Como testar a rede**
Será empregado um playbook do Ansible (`ping-alunos.yml`) a partir da máquina mestre para despachar pacotes ICMP (Ping) recursivos validando que todas as 30 estações estão na grade; além de medições com iPerf3 para garantir a taxa de transferência Gigabit.  O tráfego de saída (internet) será verificado via pacotes TCP simulando chamadas à API do GitHub e aos servidores DNS do Google.

**Possíveis problemas e falhas:**
- **Falha de Conexão Física:** Estações constando como inoperantes ou conectando em 100Mbps de forma falha.
- **IP Duplicado ou Exaustão de Leases:** O esgotamento do pool DHCP após a chegada de eventuais laptops dos próprios estudantes para uso da sala, gerando conflito lógico.

**Possíveis Soluções:**
- Reprogramar terminações de tomadas (patch cord e keystone) das máquinas que atestarem perdas de pacotes passando em um novo teste pelo equipamento certificador Cat6.
- Adequação do servidor DHCP reduzindo o tempo de aluguel (*Lease Time*) do IP de 24 horas para 2 horas e ativação de *DHCP Snooping* no core Switch, barrando roteadores clandestinos. Amarração MAC-IP estático (Reservation) poderá ser empregada para as 30 máquinas do campus. 

---

## 5. Finalização do Projeto: Formalização do fim da fase ou projeto

**Se o projeto atingiu os objetivos:**
Trabalho integralmente homologado. A infraestrutura apresentou estabilidade impecável durante os testes de estresse em que o Proxmox alocou 30 contêineres simultâneos (via Terraform), sem quedas de comunicação.

**Benefícios para a empresa (Instituição):**
O IFRO agora dispõe de um ambiente profissional, padronizado com as indústrias e resiliente para o ensino moderno. As aulas decorrem num canal blindado à interferência nas demais redes do bloco laboratorial, mitigando reclamações de indisponibilidade ou lentidão geral na sede.

**Sugestões de melhoria:**
- Implementação de um protocolo de Autenticação na porta do Switch (802.1X/Radius), em que somente dispositivos certificados via credencial estudantil consigam obter o link ativo.
- Disponibilidade de um link de fibra redundante (Failover) de secundária provedora visando a mitigação de "cair" a API externa em dias de avaliações.

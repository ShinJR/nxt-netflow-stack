# nxt-netflow-stack (NextHop Flow Stack)

> ⭐ **Se este repositório/solução lhe foi útil, não deixe de dar um "star" no projeto no GitHub.**

Stack de **NetFlow** para ISPs, baseada em **Elastic (Elasticsearch + Kibana)** + **Filebeat (módulo NetFlow)** + **Grafana**, desenvolvida pela **NextHop Solutions** através de seu CEO **Elizandro Pacheco** como contribuição para a comunidade.

🎤 Esta stack foi preparada para apresentação na:

- **Evento**: [15ª Semana de Infraestrutura da Internet no Brasil](https://semanainfra.nic.br/)
- **GTER/GTS**: [GTER | GTS (Semana de Infraestrutura)](https://gtergts.nic.br/)

---

## 📌 Sumário

- [🚀 Quick start](#quick-start)
- [🧱 Componentes](#componentes)
- [🧭 Portas e endpoints](#portas-e-endpoints)
- [🗺️ Arquitetura](#arquitetura)
- [✅ Pré-requisitos](#pre-requisitos)
- [🔧 Configuração (.env)](#configuracao-env)
- [📡 Exportadores NetFlow (equipamentos)](#exportadores-netflow)
- [🗃️ Persistência e dados](#persistencia-e-dados)
- [🧰 Operação](#operacao)
- [🔐 Segurança (produção)](#seguranca-producao)
- [🧯 Troubleshooting](#troubleshooting)
- [🤝 Apoiadores](#apoiadores)
- [📄 Licença](#licenca)

---

<a id="quick-start"></a>
## 🚀 Quick start

1) Crie seu `.env`:

```bash
cp env.example .env
```

2) Suba a stack:

```bash
docker compose up -d
```

3) Acesse:

- **Kibana**: `http://localhost:5601` (usuário `elastic`)
- **Grafana**: `http://localhost:3000` (usuário `admin`)

4) Configure seus equipamentos para exportar NetFlow para **UDP/2055** no IP do servidor.

---

<a id="componentes"></a>
## 🧱 Componentes

- **Elasticsearch (8.12.2)**: armazenamento e busca (`http://localhost:9200`)
- **Kibana (8.12.2)**: exploração/consulta (`http://localhost:5601`)
- **Filebeat (8.12.2)**: recepção e parse de **NetFlow** + envio ao Elasticsearch (**UDP/2055**)
- **Grafana (11.2.0)**: dashboards (`http://localhost:3000`)

---

<a id="portas-e-endpoints"></a>
## 🧭 Portas e endpoints

- **9200/TCP**: Elasticsearch (HTTP)
- **5601/TCP**: Kibana (UI)
- **3000/TCP**: Grafana (UI)
- **2055/UDP**: NetFlow (entrada no Filebeat)

> 💡 Dica: em produção, **não exponha** `9200/5601/3000` na Internet. Veja [Segurança](#seguranca-producao).

---

<a id="arquitetura"></a>
## 🗺️ Arquitetura

1. Seus roteadores/BRAS/switches exportam NetFlow para o **IP do servidor** desta stack na porta **UDP 2055**.
2. O **Filebeat (módulo netflow)** recebe e decodifica templates/fluxos.
3. O Filebeat envia eventos para o **Elasticsearch**.
4. Você analisa no **Kibana** e/ou monta dashboards no **Grafana**.

---

<a id="pre-requisitos"></a>
## ✅ Pré-requisitos

- **Docker** instalado
- **Docker Compose** (plugin `docker compose`)
- Host recomendado:
  - **2 vCPU** (mínimo)
  - **4 GB RAM** (recomendado)
  - Disco conforme retenção desejada (NetFlow cresce rápido)

### Linux: ajuste de kernel para Elasticsearch (recomendado)

Em muitos ambientes Linux, o Elasticsearch precisa de `vm.max_map_count` maior:

```bash
sudo sysctl -w vm.max_map_count=262144
```

Para persistir após reboot (Linux), adicione em `/etc/sysctl.conf`:

```bash
vm.max_map_count=262144
```

---

<a id="configuracao-env"></a>
## 🔧 Configuração (.env)

O `docker-compose.yaml` lê variáveis do `.env`. O repositório inclui `env.example` como base.

### Variáveis principais

- **Elasticsearch**: `ELASTIC_PASSWORD` (usuário `elastic`)
- **Kibana (usuário interno)**: `KIBANA_SYSTEM_PASSWORD` (usuário `kibana_system`)
- **Grafana**: `GRAFANA_ADMIN_USER` e `GRAFANA_ADMIN_PASSWORD`

### Por que o `.env` é obrigatório?

Por segurança, o `docker compose` **falha** se você não definir as variáveis (para evitar subir com credenciais em branco).

### Senha do `kibana_system` (automação)

O container `setup_kibana_system_password` roda uma vez e **configura a senha** do usuário `kibana_system` no Elasticsearch usando `KIBANA_SYSTEM_PASSWORD`.

---

<a id="exportadores-netflow"></a>
## 📡 Exportadores NetFlow (equipamentos)

Configure seus roteadores/BRAS/switches para exportar:

- **Destino**: IP do servidor desta stack
- **Porta**: **2055/UDP**

Compatibilidade comum:

- **NetFlow v9 / IPFIX**: recomendado (templates + flexibilidade)
- **NetFlow v5**: também funciona em muitos cenários

> 🧠 Se o host estiver atrás de firewall/NAT, garanta a liberação/encaminhamento de **UDP/2055** até o servidor Docker.

### MikroTik / RouterOS (Traffic Flow → NetFlow v9)

Exemplo via terminal (ajuste interfaces, IP e timeouts conforme seu cenário):

```shell
/ip traffic-flow set enabled=yes interfaces=all cache-entries=4k active-flow-timeout=30m inactive-flow-timeout=15s
/ip traffic-flow target add dst-address=<IP_DO_SERVIDOR_FILEBEAT> port=2055 version=9
```

Referência: documentação oficial do RouterOS **Traffic Flow** em [MikroTik — Traffic Flow](https://help.mikrotik.com/docs/spaces/ROS/pages/21102653/Traffic%2BFlow).

> 💡 Dica: se você já tiver um target criado, revise com `/ip traffic-flow target print` para evitar duplicar targets.

### Huawei VRP (NetStream → NetFlow v9)

Exemplo (os comandos podem variar por versão de VRP/modelo; ajuste interface e timeouts):

```shell
system-view
ip netstream export version 9
ip netstream export host <IP_DO_SERVIDOR_FILEBEAT> 2055
ip netstream export source <INTERFACE_OU_LOOPBACK_DE_ORIGEM>
ip netstream timeout active 1
ip netstream timeout inactive 15

interface <INTERFACE_MONITORADA>
 ip netstream inbound
 ip netstream outbound
quit
save
```

Referências:

- Guia de configuração de exportação NetStream (exemplo): [ManageEngine — Configuring NetStream Export (Huawei)](https://www.manageengine.com/products/netflow/help/configuring-netstream-export.html)
- Exemplo oficial de exportação de estatísticas/flow (Huawei): [Huawei Support — Example for Configuring Flexible Flow Statistics Export](https://support.huawei.cn/enterprise/en/doc/EDOC1100468733/85a6c438/example-for-configuring-flexible-flow-statistics-export)

### Filebeat (módulo NetFlow)

Para entender o comportamento/formatos suportados (NetFlow v5/v9/IPFIX), referência: [Elastic — Filebeat NetFlow module](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-module-netflow.html).

---

<a id="persistencia-e-dados"></a>
## 🗃️ Persistência e dados

Os dados persistem em volumes Docker:

- Elasticsearch: `nexthop_flow_stack_elasticsearch_data`
- Grafana: `nexthop_flow_stack_grafana_data`

Listagem rápida:

```bash
docker volume ls | grep nexthop_flow_stack || true
```

---

<a id="operacao"></a>
## 🧰 Operação

### Subir / atualizar containers

```bash
docker compose up -d
```

### Ver status e logs

```bash
docker compose ps
docker compose logs -f
```

### Parar / iniciar

```bash
docker compose stop
docker compose start
```

### Derrubar (mantendo dados)

```bash
docker compose down
```

### Derrubar e apagar dados (cuidado!)

```bash
docker compose down -v
```

---

<a id="seguranca-producao"></a>
## 🔐 Segurança (produção)

- **Não exponha** `9200`, `5601` e `3000` diretamente na Internet.
- Publique UI somente via **VPN** (WireGuard, etc.) ou reverse proxy com **TLS** + autenticação.
- **Gere/rotacione segredos** do `.env` e não compartilhe o arquivo.
- Restrinja quem pode enviar para **UDP/2055** (ACLs/VRF/VLAN/Firewall).

---

<a id="troubleshooting"></a>
## 🧯 Troubleshooting

### Elasticsearch não sobe / reinicia em loop

- Verifique `vm.max_map_count` (Linux) e recursos do host (RAM/CPU/disco).
- Logs:

```bash
docker compose logs -f elasticsearch
```

### Kibana não conecta no Elasticsearch

- Confirme se o `setup_kibana_system_password` rodou com sucesso:

```bash
docker compose logs setup_kibana_system_password
```

- Confira se `KIBANA_SYSTEM_PASSWORD` no `.env` é o mesmo que o setup aplicou no Elasticsearch.

### Não chega NetFlow

- Confirme se a porta está aberta no host (**UDP/2055**) e se o exporter aponta para o IP correto.
- Logs:

```bash
docker compose logs -f filebeat
```

### Health check do Elasticsearch retorna 401

Isso é esperado: o Elasticsearch está com security habilitada; **401 também indica que ele está respondendo**.

---

## 📊 Grafana (recursos)

- **Template de dashboard (JSON)**: [`template-grafana.json`](./template-grafana.json)
- **Apresentação na GTER (PDF)**: [`[GTER 54] Elizandro Pacheco.pdf`](./%5BGTER%2054%5D%20Elizandro%20Pacheco.pdf)

<a id="apoiadores"></a>
## 🤝 Apoiadores

Obrigado aos apoiadores que fortalecem a iniciativa e a comunidade:

### NextHop Solutions

<a href="https://nexthop.solutions/">
  <img
    src="https://nexthop.solutions/wp-content/uploads/elementor/thumbs/NEXTHOP-LATERAL-1-r05q2h0ahcqhnnc4r4rn53b72q7bg3wzp3dnxbh7nk.png"
    alt="NextHop Solutions"
    width="260"
  />
</a>

- Site: `https://nexthop.solutions/`

### EvoCODE IA

<a href="https://evocode.dev.br/">
  <img
    src="https://evocode.dev.br/lovable-uploads/c52c93b9-8fbf-4e6d-998d-46475b3ede3d.png"
    alt="EvoCODE IA"
    width="260"
  />
</a>

- Site: `https://evocode.dev.br/`

---

## 📫 Contato

- **Desenvolvedor**: Elizandro Pacheco
- **Instagram**: [`@elizandropacheco`](https://instagram.com/elizandropacheco)
- **WhatsApp**: [`+55 51 99871-8111`](https://wa.me/5551998718111)

<a id="licenca"></a>
## 📄 Licença

Este projeto é distribuído sob a licença **Apache 2.0**. Veja o arquivo `LICENSE`.

### Resumo simples (o que pode e o que não pode) 🔎

> Nota: este é um resumo prático e **não** substitui a leitura do `LICENSE` (nem é aconselhamento jurídico).

#### Você PODE ✅

- **Usar** a stack/código para qualquer finalidade (inclusive **comercial**).
- **Modificar** (criar derivações/forks), inclusive “rebatizar” o seu fork e manter outro mantenedor.
- **Redistribuir** (publicar) o original ou versões modificadas (em código-fonte ou empacotado).

#### Você DEVE 📌

- **Incluir uma cópia da licença Apache 2.0** quando redistribuir.
- **Manter avisos de copyright/atribuição** e notices existentes.
- **Indicar mudanças**: arquivos modificados devem conter aviso claro de que foram alterados.
- Se houver arquivo `NOTICE` no futuro, **reproduzir o conteúdo aplicável** ao redistribuir.

#### Você NÃO PODE 🚫

- **Usar marcas/nome/logotipos** da NextHop Solutions (ou de terceiros) como se houvesse endosso/afiliações: a Apache 2.0 **não concede licença de marca** (ver “Trademarks” no `LICENSE`).
- **Remover atribuições/avisos legais** existentes do projeto original ao redistribuir.

Em outras palavras: **sim**, a Apache 2.0 permite você usar/alterar/redistribuir e até trocar o mantenedor do *seu fork*, **desde que** você cumpra as obrigações de atribuição/licença e não use marcas como se fossem permissão/endorso.

### Licença em PT-BR (referência) 🇧🇷

Para facilitar leitura, existe também o arquivo `LICENSE.pt-BR.md` (referência). Em caso de dúvida legal, use sempre o `LICENSE` (EN).

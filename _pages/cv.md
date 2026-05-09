---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Experiência
======

**V360 — SRE Tech Lead** (Jan 2026 – Presente) *Remoto*
* Liderança de iniciativas de SRE com foco em confiabilidade, observabilidade, automação e excelência operacional para sistemas mission-critical
* Responsável pela manutenção de targets de SLA/SLO, garantindo latência web abaixo de 500ms e processamento de filas abaixo de 30 segundos
* Gerenciamento de ambientes EKS e ECS em produção, suportando serviços críticos e times de engenharia distribuídos
* Liderança na otimização de custos de infraestrutura migrando workloads de AWS Lambda para Kubernetes, reduzindo custos mensais de cloud em 83% (de USD 7k para USD 1.2k)
* Projeto e manutenção de runners self-hosted do GitHub Actions na AWS, reduzindo tempo de execução de pipelines CI/CD de horas para aproximadamente 15 minutos
* Implementação de soluções de observabilidade, dashboards operacionais e sistemas de alertas usando CloudWatch e OpenSearch
* Liderança na integração do IAM Identity Center e deploy de Keycloak em Kubernetes para autenticação centralizada e governança de acesso
* Suporte a resposta a incidentes, análise de causa raiz (RCA) e troubleshooting de sistemas distribuídos de alta disponibilidade

**V360 — Cloud Engineer (Pleno)** (Jan 2025 – Mai 2026) *Remoto*
* Liderança na modernização de infraestrutura migrando workloads de AWS Elastic Beanstalk para ECS, incluindo containerização Docker e design de infraestrutura cloud-native
* Otimização de performance da plataforma, reduzindo latência de aplicações web de aproximadamente 1.5s para 600ms através de melhorias arquiteturais e de infraestrutura
* Redesign da arquitetura de processamento de filas, dividindo workers monolíticos em múltiplos serviços dedicados, melhorando significativamente a escalabilidade e latência operacional
* Execução de upgrades de versões major do PostgreSQL (v14 para v17) em ambientes de produção com downtime mínimo e validação de disaster recovery
* Validação de backup e recuperação do PostgreSQL e testes de disaster recovery para garantir confiabilidade operacional
* Implementação de automação e melhorias de CI/CD usando GitHub Actions e runners self-hosted na AWS

**V360 — Cloud Engineer (Júnior) & Estagiário de Infraestrutura** (Nov 2022 – Jan 2025) *Remoto*
* Suporte a operações de produção AWS, automação de deploy, monitoramento de infraestrutura e manutenção de ambientes cloud
* Auxílio na configuração de monitoramento com CloudWatch, suporte operacional e iniciativas de confiabilidade de infraestrutura

Formação
======
* **M.Sc. em Ciência da Computação** — Universidade Federal de Pelotas (UFPel), 2022 – 2025
* **Eng. em Engenharia de Energia** — Universidade Federal de Santa Catarina (UFSC), 2013 – 2021

Habilidades Técnicas
======
* **Cloud & Orquestração:** AWS (EKS, ECS, EC2, Lambda, RDS, OpenSearch, Route53, WAF, IAM Identity Center, VPC, CloudWatch), Kubernetes, Docker
* **Monitoramento & Confiabilidade:** CloudWatch, OpenSearch, Observabilidade, Resposta a Incidentes, SLA/SLO, PostgreSQL, Redis
* **CI/CD & Automação:** GitHub Actions, Self-hosted Runners, Automação de Infraestrutura
* **Programação & Scripting:** Bash, Ruby, Python, SQL
* **Sistemas & Segurança:** Linux, IAM Identity Center, Keycloak, Gerenciamento de Secrets
* **Conceitos:** SRE, Platform Engineering, Alta Disponibilidade, Sistemas Distribuídos, Excelência Operacional

Certificações
======
* AWS Educate – Introduction to Cloud
* Google Cloud Computing Foundations
* Practical Data Science

Idiomas
======
* **Português:** Nativo
* **Inglês:** Proficiência Profissional
* **Espanhol:** Proficiência Básica

Publicações
======
  <ul>{% for post in site.publications reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>

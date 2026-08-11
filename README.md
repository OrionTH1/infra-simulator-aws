# infra-simulator-aws

Um simulador que roda no navegador e mostra como a infraestrutura do [`infra-terraform-aws`](https://github.com/OrionTH1/infra-terraform-aws) se comporta sob carga. Você liga usuários no load balancer, aumenta a taxa de requisições, e vê o autoscaling entrar, a latência subir enquanto a task nova não fica saudável, o WAF barrar um IP que passou do limite, e o Aurora escalar ACU.

Nada aqui toca a AWS. É tudo simulação matemática no cliente, sem backend e sem custo.

**[Abrir o simulador](https://orionth1.github.io/infra-simulator-aws/)**

[![O simulador em execução](.github/assets/demo.gif)](https://orionth1.github.io/infra-simulator-aws/)

## Por que ele existe

Manter uma infraestrutura AWS de pé 24 horas por dia só para poder mostrá-la é caro, e um diagrama estático não mostra a parte que interessa. As decisões dessa arquitetura aparecem no comportamento: quanto tempo passa entre o pico de tráfego e a capacidade nova estar servindo, o que acontece com as requisições enquanto isso, por que o rate limit por IP não protege contra um ataque distribuído.

## O que ele modela

- **Tráfego** de cards de usuário conectados ao ALB, com taxa configurável e padrões constante, rampa ou burst
- **WAF** com rate limit por IP, contando em janela deslizante, e a resposta 403 voltando para quem foi barrado
- **Distribuição** do ALB entre as tasks saudáveis, incluindo o comportamento de fail open quando não sobra nenhuma
- **Ciclo de vida da task** do provisionamento até ficar saudável, com o pull da imagem passando pelo ECR e pelo S3, e a busca do segredo antes do container subir
- **Autoscaling** por requisições por target, com as janelas de avaliação e o cooldown separados por direção
- **Aurora** com writer e reader, divisão de leitura e escrita, cache de buffer, escala de ACU e failover quando uma instância morre
- **Alarmes do CloudWatch** com a semântica real de período, estatística, `evaluation_periods` e `treat_missing_data`

Cada pacote na tela é uma requisição percorrendo o caminho inteiro, ida e volta.

## Calibração

Este é o ponto que separa o simulador de uma animação bonita: **todo número tem origem declarada**. São três, e apenas três:

1. O Terraform do repositório de infraestrutura, quando o valor está lá e muda se ele mudar
2. Documentação ou benchmark publicado da AWS, com a aritmética da derivação explícita
3. Premissa declarada sobre o workload, quando não existe fonte

O [`CALIBRATION.md`](CALIBRATION.md) lista item por item, e tem uma seção chamada "Números que ainda não têm fonte" onde ficam registrados os que foram escolhidos por ritmo de demonstração em vez de medição. Preferi assumir isso a apresentar tudo como se fosse medido.

## Rodar local

```bash
npm install
npm run dev
```

```bash
npm test     # 318 testes
npm run lint
npm run build
```

Os testes cobrem a simulação, não a interface: distribuição de tráfego, janela do rate limit, transições do ciclo de vida da task, capacidade do Aurora, avaliação dos alarmes e a geometria do canvas.

## Stack

React com TypeScript e Vite, [React Flow](https://reactflow.dev) para o canvas, Zustand para o estado da simulação e Tailwind. O deploy sai pelo GitHub Pages a cada push na branch principal.

## O que ele não faz

Não aplica nada, não lê nada da AWS e não conhece nenhuma conta. Também não simula o que a infraestrutura do outro repositório não tem: não existe bucket da aplicação, o S3 que aparece no desenho é onde o ECR guarda as camadas da imagem.

O [`SIMULATOR_PLAN.md`](SIMULATOR_PLAN.md) guarda o backlog, o que já foi feito, e a lista do que foi recusado com o motivo.

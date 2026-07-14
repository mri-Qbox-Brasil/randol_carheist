# randol_carheist — Manual

Roubo de carro sob encomenda: pegue o serviço com o NPC, invada a garagem marcada, despiste o rastreador da polícia por 5 minutos, entregue o veículo e troque os papéis por dinheiro.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Item `heist_papers`](#item-heist_papers)
4. [Configuração](#configuração)
5. [Fluxo do roubo](#fluxo-do-roubo)
6. [Rastreador e polícia](#rastreador-e-polícia)
7. [Cooldown](#cooldown)
8. [Bridge de framework](#bridge-de-framework)
9. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
10. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `ox_lib` | Sim | Callbacks, `lib.points`, zonas, Text UI, `progressCircle` |
| Framework (`qb-core`, `es_extended` ou `ND_Core`) | Sim | Um deles. A bridge detecta automaticamente qual está rodando |
| `qb-target` | Sim | Usado direto no cliente (NPC, entrada da garagem), sem alternativa configurável |
| Inventário (`ox_inventory` ou `qb-inventory`) | Sim | O item `heist_papers` guarda o valor da recompensa em metadata |
| `ps-dispatch` | Não | O bridge do QBCore chama `exports['ps-dispatch']:VehicleTheft(vehicle)` em `AlertPolice`. Se a função não existir na bridge, o alerta é simplesmente pulado |

---

## Instalação

1. Copie a pasta `randol_carheist` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure randol_carheist
   ```
3. Cadastre o item `heist_papers` no seu inventário (ver seção abaixo) e copie `image/heist_papers.png` para a pasta de imagens do inventário.
4. Ajuste `RequiredCops`, `Cooldown` e a faixa de pagamento em `sv_config.lua`.

---

## Item `heist_papers`

O recurso entrega o item `heist_papers` com metadata (`amount`, `cid`, `description`) no lugar de pagar direto. Só quem roubou o carro consegue trocar os papéis por dinheiro — o servidor compara o `cid` da metadata com o do jogador.

Para `ox_inventory`, em `data/items.lua`:

```lua
['heist_papers'] = {
    label = 'Vehicle Papers',
    weight = 0,
    stack = false,
    close = true,
    description = 'Delivery documents.',
    client = {
        image = 'heist_papers.png',
    }
},
```

Para `qb-inventory`, em `shared/items.lua`:

```lua
heist_papers = {
    name = 'heist_papers',
    label = 'Vehicle Papers',
    weight = 0,
    type = 'item',
    image = 'heist_papers.png',
    unique = true,
    useable = false,
    shouldClose = true,
    combinable = nil,
    description = 'Delivery documents.'
},
```

---

## Configuração

Arquivo: `sv_config.lua`. O servidor envia a config inteira para os clientes ao iniciar o recurso e ao jogador logar (evento `randol_carheist:cacheConfig`).

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Debug` | bool | Não | Desenha a zona de entrega e coloca um waypoint automático na garagem sorteada |
| `RequiredCops` | number | Sim | Mínimo de policiais em serviço (job do tipo `leo`, on duty) para o serviço ser aceito. Padrão: `0` |
| `PedModel` | hash | Sim | Modelo do NPC contratante. Padrão: `U_M_Y_SmugMech_01` |
| `PedCoords` | vec4 | Sim | Posição e heading do NPC. O ped só é criado a 50 m de distância |
| `FuelScript.enable` | bool | Sim | `true` chama `exports[FuelScript.name]:SetFuel(veh, 100.0)`. `false` define `Entity(veh).state.fuel = 100` |
| `FuelScript.name` | string | Sim | Nome do recurso de combustível. Padrão: `LegacyFuel` |
| `DeliveryCoords` | array de vec3 | Sim | Pontos de entrega possíveis. Um é sorteado por serviço. Padrão: 3 pontos |
| `Min` / `Max` | number | Sim | Faixa do pagamento, sorteada na entrega. Padrão: `6500` a `8700` |
| `Cooldown` | number (min) | Sim | Prazo do serviço **e** cooldown global do recurso. Padrão: `30` |
| `VehicleList` | array de strings | Sim | Modelos possíveis do carro alvo. Um é sorteado por serviço. Padrão: 17 supercarros |
| `SpawnLocations` | array de `{ enter: vec4, spawn: vec4 }` | Sim | Garagens. `enter` é onde aparece a interação de abrir o portão; `spawn` é onde o carro nasce. Padrão: 7 garagens |

---

## Fluxo do roubo

1. O NPC (`PedCoords`) tem duas opções de target: **Start Job** e **Return Papers**.
2. Ao aceitar, o servidor sorteia modelo, garagem e ponto de entrega. Um blip de raio de 100 m marca a região da garagem — a posição exata não é revelada.
3. Chegando na garagem, a opção "Enter Garage" toca a animação de abrir o portão, escurece a tela e spawna o carro sorteado com o jogador já no banco do motorista. Placa `CARH` + 4 dígitos, cores aleatórias, tanque cheio.
4. O rastreador liga: por 5 minutos a polícia recebe a posição do carro a cada 5 segundos.
5. O ponto de entrega é marcado no GPS. **A entrega só é aceita com o rastreador desligado.**
6. Na zona de entrega, `E` inicia a descarga (3 s). O servidor confere que o jogador está a menos de 10 m do ponto, deleta o carro e entrega os `heist_papers` com o valor sorteado.
7. De volta ao NPC, "Return Papers" troca o item pelo dinheiro em espécie.

Se o jogador cair do servidor ou deslogar durante o serviço, o carro é deletado e o roubo é cancelado.

---

## Rastreador e polícia

Enquanto o rastreador está ativo, o servidor chama `PoliceTracker(coords)` a cada 5 segundos, que envia `randol_carheist:client:trackerUpdate` para todos os policiais em serviço. Do lado deles, um blip pulsante "CAR HEIST" acompanha o carro.

O rastreador roda 60 ciclos de 5 segundos (5 minutos) e então dispara `randol_carheist:client:trackerOff`, avisando o ladrão de que está liberado para entregar. Se o carro deixar de existir antes disso, o serviço é cancelado.

A contagem de policiais (`CheckCopCount`) e o alerta inicial (`AlertPolice`) vêm da bridge. No QBCore, o alerta usa `ps-dispatch`.

---

## Cooldown

`Cooldown` (em minutos) tem duas funções ao mesmo tempo:

- É o **prazo** do serviço. Quando expira, o roubo é resetado: o jogador recebe "You ran out of time, ditch the car" e o carro perde a utilidade (a entrega não é mais aceita).
- É o **cooldown global do servidor**. Enquanto ele estiver correndo, nenhum outro jogador consegue pegar o serviço ("Heist is on cooldown").

Ou seja, apenas um roubo por vez roda no servidor, e o próximo só libera `Cooldown` minutos depois do anterior ter começado.

---

## Bridge de framework

Cada arquivo em `bridge/` só é carregado se o recurso correspondente estiver `started`:

| Framework | Arquivos | Detecção |
|---|---|---|
| QBCore | `bridge/client/qb.lua`, `bridge/server/qb.lua` | `qb-core` |
| ESX | `bridge/client/esx.lua`, `bridge/server/esx.lua` | `es_extended` |
| ND | `bridge/client/nd.lua`, `bridge/server/nd.lua` | `ND_Core` |

Funções fornecidas pela bridge do servidor: `GetPlayer`, `DoNotification`, `GetPlyIdentifier`, `GetSourceFromIdentifier`, `GetCharacterName`, `GetItemData`, `RemoveHeistPapers`, `AddHeistPapers`, `AddRewardMoney`, `CheckCopCount`, `PoliceTracker`.

Funções da bridge do cliente: `handleVehicleKeys`, `hasPlyLoaded`, `DoNotification` e `AlertPolice` (opcional — só é chamada se estiver definida).

O bridge do QBCore detecta o `ox_inventory` automaticamente e usa `GetSlotWithItem` / `AddItem` com metadata; sem ele, cai no `qb-inventory`.

---

## Entrypoints para outros recursos

Não há exports. Callbacks registrados no servidor:

```lua
-- Valida polícia e cooldown; sorteia modelo, garagem e entrega. Retorna a tabela do serviço ou false
lib.callback.await('randol_carheist:attemptjob', false)

-- Spawna o carro na garagem, liga o rastreador e alerta a polícia. Retorna o netId
lib.callback.await('randol_carheist:server:createVehicle', false)

-- Valida a distância da entrega (10 m), deleta o carro e entrega os heist_papers
lib.callback.await('randol_carheist:server:finishHeist', false, vehicleNetId)

-- Troca os heist_papers pelo dinheiro (valida o cid da metadata)
lib.callback.await('randol_carheist:server:returnPapers', false)
```

Eventos de cliente disparados pelo servidor (todos ignoram chamadas vindas de outros recursos):

| Evento | Alvo | Descrição |
|---|---|---|
| `randol_carheist:cacheConfig` | Todos | Envia a config do servidor ao cliente |
| `randol_carheist:client:trackerUpdate` | Polícia | Posição atual do carro roubado |
| `randol_carheist:client:trackerOff` | Todos | Rastreador expirou |
| `randol_carheist:client:resetHeist` | Todos | Prazo esgotado, serviço cancelado |
| `randol_carheist:client:endRobbery` | Todos | Entrega concluída |

---

## Estrutura de arquivos

```
randol_carheist/
├── cl_carheist.lua       — NPC, blip de área, entrada da garagem, blips do rastreador, entrega
├── sv_carheist.lua       — sorteio do serviço, spawn do carro, rastreador, cooldown, papéis e pagamento
├── sv_config.lua         — polícia mínima, NPC, combustível, entregas, pagamento, carros, garagens
├── bridge/
│   ├── client/
│   │   ├── qb.lua        — QBCore: chaves, notificações, alerta ps-dispatch
│   │   ├── esx.lua       — ESX
│   │   └── nd.lua        — ND
│   └── server/
│       ├── qb.lua        — QBCore: player, inventário (ox/qb), contagem de policiais, tracker
│       ├── esx.lua       — ESX
│       └── nd.lua        — ND
├── image/
│   └── heist_papers.png  — ícone do item para o inventário
└── fxmanifest.lua
```

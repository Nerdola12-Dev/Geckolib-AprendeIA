# AprendeIA — Geckolib 5 (para Minecraft 1.21.5+)

> **Documento de aprendizagem gerado pelo "gerrador" (IA).**

---

## Objetivo
Criar um material de aprendizagem claro, prático e progressivo sobre **Geckolib 5** (compatível com Minecraft 1.21.5 e superiores), cobrindo desde conceitos básicos até integração com **Forge 1.21.8** e exemplos práticos.

## Público-alvo
Desenvolvedores intermediários a avançados familiarizados com Java e com os conceitos básicos de modding para Minecraft (noções de Forge e MCreator não são obrigatórias, mas ajudam).

## Pré-requisitos
- Java 17+ (ou versão exigida pelo Forge/MC). 
- Ambiente de desenvolvimento (IntelliJ/VS Code/Eclipse).
- Noções de Gradle e do ciclo de desenvolvimento de mods.
- Familiaridade básica com as APIs do Minecraft/Forge.

---

# 📌 Instalação do Geckolib 5 (1.21.5+)

### Repositório Maven

No seu `build.gradle`:

```gradle
repositories {
    exclusiveContent {
        forRepository {
            maven {
                name = 'GeckoLib'
                url = 'https://dl.cloudsmith.io/public/geckolib3/geckolib/maven/'
            }
        }
        filter { 
            includeGroup('software.bernie.geckolib')
        }
    }
}
```

### Dependências

**Fabric:**

```gradle
dependencies {
    modImplementation "software.bernie.geckolib:geckolib-fabric-${minecraft_version}:${geckolib_version}"
}
```

**Forge:**

```gradle
dependencies {
    implementation fg.deobf("software.bernie.geckolib:geckolib-forge-${minecraft_version}:${geckolib_version}")
}
```

**NeoForge:**

```gradle
dependencies {
    implementation "software.bernie.geckolib:geckolib-neoforge-${minecraft_version}:${geckolib_version}"
}
```

### Mixins Plugin

**settings.gradle:**

```gradle
pluginManagement {
    repositories {
        maven { url = 'https://repo.spongepowered.org/repository/maven-public/' }
    }
}
```

**build.gradle:**

```gradle
plugins {
    id 'org.spongepowered.mixin' version '0.7.+'
}
```

### Gradle Properties

No `gradle.properties`:

```properties
geckolib_version=???  # Coloque a versão mais recente
```

### Observação Multiloader

Todo o código está em `common` e pode ser usado em qualquer loader sem limitações.

---

# 🦎 Geo Models no Geckolib 5

### Conceito

Geo Models conectam:

- Arquivos `.geo.json` (modelos)
- Arquivos `.animation.json` (animações)
- Texturas `.png`

Todo objeto animável precisa de um Geo Model.

### Criando um Geo Model

1. Crie uma classe que estenda `GeoModel<T>`.
2. Implemente os métodos:

- `getModelResource` → caminho do modelo JSON
- `getTextureResource` → caminho da textura
- `getAnimationResource` → caminho do arquivo de animação

### DefaultedGeoModel (recomendado)

```java
public class ExampleModel extends DefaultedEntityGeoModel<ExampleEntity> {
    public ExampleModel() {
        super(ResourceLocation.fromNamespaceAndPath(MyMod.MOD_ID, "example"));
    }
}
```

Arquivos procurados:

- Modelo: `assets/<modid>/geckolib/models/entity/example.geo.json`
- Animação: `assets/<modid>/geckolib/animations/entity/example.animation.json`
- Textura: `assets/<modid>/textures/entity/example.png`

### BaseGeoModel (personalizado)

```java
public class ExampleModel extends GeoModel<ExampleItem> {
    private final ResourceLocation model = ResourceLocation.fromNamespaceAndPath(MyMod.MOD_ID, "example");
    private final ResourceLocation animations = ResourceLocation.fromNamespaceAndPath(MyMod.MOD_ID, "example");
    private final ResourceLocation texture = ResourceLocation.fromNamespaceAndPath(MyMod.MOD_ID, "textures/item/example.png");

    @Override
    public ResourceLocation getModelLocation(GeoRenderState renderState) { return this.model; }
    @Override
    public ResourceLocation getAnimationResource(ExampleItem animatable) { return this.animations; }
    @Override
    public ResourceLocation getTextureLocation(GeoRenderState renderState) { return this.texture; }
}
```

### Estrutura de Arquivos

- Modelos `.geo.json`: `resources/assets/<modid>/geckolib/models/`
- Animações `.animation.json`: `resources/assets/<modid>/geckolib/animations/`
- Texturas `.png`: `resources/assets/<modid>/textures/`

---

# 🧩 Criando Entidades com GeckoLib

A criação de uma entidade animada com GeckoLib segue alguns passos específicos. 

## Passos rápidos
1. Criar o **modelo no Blockbench**.
2. Criar o **Geo Model** (já explicado na seção anterior).
3. Criar a **classe da entidade** implementando `GeoEntity`.
4. Criar e registrar o **renderer** usando `GeoEntityRenderer`.

Os passos #1 e #2 já foram detalhados em suas próprias seções. Aqui vamos focar nos passos #3 e #4.

---

## A Classe da Entidade

Uma entidade GeckoLib precisa:
- Implementar **`GeoEntity`**.
- Criar uma instância de **`AnimatableInstanceCache`** via `GeckoLibUtil.createInstanceCache(this)`.
- Implementar o método `getAnimatableInstanceCache` retornando a instância criada.
- Implementar `registerControllers`, onde são definidos os controladores de animações.

### Exemplo — Classe de Entidade
```java
public class ExampleEntity extends PathfinderMob implements GeoEntity {
    protected static final RawAnimation FLY_ANIM = RawAnimation.begin().thenLoop("move.fly");

    private final AnimatableInstanceCache geoCache = GeckoLibUtil.createInstanceCache(this);

    public ExampleEntity(EntityType<? extends ExampleEntity> type, Level level) {
        super(type, level);
    }

    @Override
    public void registerControllers(final AnimatableManager.ControllerRegistrar controllers) {
        controllers.add(new AnimationController<>("Flying", 5, this::flyAnimController));
    }

    protected <E extends ExampleEntity> PlayState flyAnimController(final AnimationTest<E> animTest) {
        if (animTest.isMoving())
            return animTest.setAndContinue(FLY_ANIM);

        return PlayState.STOP;
    }

    @Override
    public AnimatableInstanceCache getAnimatableInstanceCache() {
        return this.geoCache;
    }
}
```

Nesse exemplo:
- Criamos uma animação simples de voo (`FLY_ANIM`).
- Usamos um **Animation Controller** para controlar o comportamento.
- A animação é aplicada quando a entidade está em movimento (`animTest.isMoving()`).

---

## O Renderer da Entidade

Diferente das entidades vanilla, aqui registramos um **`GeoEntityRenderer`** ao invés de um `EntityRenderer` comum.

### Vantagens:
- Não é necessário registrar **model layers** ou **mesh definitions**.
- Basta passar o `GeoModel` e o contexto do renderizador.

### Exemplo — Renderer de Entidade
```java
public class ExampleEntityRenderer<R extends LivingEntityRenderState & GeoRenderState> 
        extends GeoEntityRenderer<ExampleEntity, R> {
    public ExampleEntityRenderer(EntityRendererProvider.Context context) {
        super(context, new ExampleEntityModel());
    }
}
```

---

## Problemas Comuns

### 🚫 Crash ao spawnar a entidade
**Erro:**
```
java.lang.NullPointerException: Cannot invoke "net.minecraft.client.renderer.entity.EntityRenderer.shouldRender(...)" because "entityrenderer" is null
```
**Causa:** O renderer da entidade não foi registrado.

---

### ⚔️ `DefaultAnimations#genericAttackAnimation` não faz a entidade atacar
- É necessário chamar explicitamente `swing()` **ou** usar um comportamento (`MeleeAttackGoal`) que chama `swing()`.
- Se sua entidade **não** estende `Monster`, você precisa sobrescrever `aiStep` e chamar `updateSwingTime()` para atualizar o tempo de ataque.

---

### ⚔️ `genericAttackAnimation` só executa parte da animação
- O tempo padrão de swing é **6 ticks** (incluindo a transição).
- Se sua animação é maior que isso, ela será cortada.
- **Solução:** sobrescreva `getCurrentSwingDuration()` na entidade e retorne um valor mais apropriado (em ticks).

---

### 🔄 Não consigo repetir a animação após executá-la
- Verifique se você resetou a animação após sua execução (veja `DefaultAnimations#genericAttackController`).
- Certifique-se de que o `loop` no arquivo `.animation.json` **não** está definido como `hold_on_last_frame`, pois isso impede a repetição.

---

## ✅ Exercício prático (Entidades com Geckolib)
1. Crie uma entidade chamada `FlyingTestEntity` que estenda `PathfinderMob` e implemente `GeoEntity`.
2. Adicione uma animação de voo (`move.fly`) em seu arquivo `.animation.json`.
3. Crie e registre um **Geo Model** (`FlyingTestEntityModel`).
4. Implemente um **renderer** com `GeoEntityRenderer`.
5. Teste no jogo: a entidade deve tocar a animação de voo enquanto se move.

---

# 🧱 Blocos Animados com GeckoLib

> **Observação:** Diferente de entidades, blocos em Minecraft são estáticos. Para animá-los com GeckoLib, **é necessário que o bloco possua um BlockEntity**, já que o sistema de renderização animada do GeckoLib depende de `BlockEntityRenderer`.

---

## Passos para criar um bloco animado

1. Criar o **modelo no Blockbench**.
2. Criar o **Geo Model** correspondente.
3. Criar a **classe do bloco** e seu **BlockEntity**.
4. Desabilitar o renderizador vanilla do bloco.
5. Criar e registrar o **renderer** para o bloco.

> Os passos 1 e 2 já são abordados nas seções de Geo Models; aqui focaremos nos passos 3 a 5.

---

## A Classe do Bloco

* O bloco deve implementar `EntityBlock` (ou uma subclasse), para que seja reconhecido como um **BlockEntity**.
* No método `newBlockEntity`, retorne uma nova instância do BlockEntity associado.

---

## A Classe BlockEntity

Para criar um **BlockEntity animável**:

1. Implemente `GeoBlockEntity`.
2. Crie um **AnimatableInstanceCache** via `GeckoLibUtil.createInstanceCache(this)`.
3. Sobrescreva `getAnimatableInstanceCache()` para retornar o cache.
4. Sobrescreva `registerControllers()` e defina suas animações.

### Exemplo — BlockEntity Animável

```java
public class ExampleBlockEntity extends BlockEntity implements GeoBlockEntity {
    protected static final RawAnimation DEPLOY_ANIM = RawAnimation.begin()
            .thenPlayOnce("misc.deploy")
            .thenLoop("misc.idle");

    private final AnimatableInstanceCache cache = GeckoLibUtil.createInstanceCache(this);

    public ExampleBlockEntity(BlockPos pos, BlockState state) {
        super(MyBlockEntities.EXAMPLE_BLOCK_ENTITY.get(), pos, state);
    }

    @Override
    public void registerControllers(AnimatableManager.ControllerRegistrar controllers) {
        controllers.add(new AnimationController<>(this::deployAnimController));
    }

    protected <E extends ExampleBlockEntity> PlayState deployAnimController(final AnimationTest<E> animTest) {
        return animTest.setAndContinue(DEPLOY_ANIM);
    }

    @Override
    public AnimatableInstanceCache getAnimatableInstanceCache() {
        return this.cache;
    }
}
```

---

## O Renderer do BlockEntity

* Crie uma classe que estenda `GeoBlockRenderer`.
* Passe o **Geo Model** e o contexto de renderização.

### Exemplo — Renderer

```java
public class ExampleBlockEntityRenderer extends GeoBlockRenderer<ExampleBlockEntity> {
    public ExampleBlockEntityRenderer(EntityRendererProvider.Context context) {
        super(new ExampleBlockEntityModel());
    }
}
```

* Depois, registre o renderer no Minecraft para que o bloco apareça animado no mundo.
* GeckoLib trata automaticamente a **direcionalidade do bloco**, mas você pode sobrescrever `rotateBlock` no `GeoBlockRenderer` para ajustes especiais.

---

## ✅ Dicas e Observações

* Não tente animar o bloco diretamente; sempre use **BlockEntity**.
* Verifique se as animações estão corretamente definidas no `.animation.json`.
* Teste o bloco no mundo para confirmar que o renderer está funcionando e que as animações estão sendo aplicadas corretamente.

# 🛡️ Itens Animados com GeckoLib

> **Observação:** Diferente de entidades e blocos, Minecraft mantém apenas **uma instância de cada Item**, então animações para itens precisam de um tratamento especial para sincronização com o lado servidor e a renderização.

---

## Passos para criar um item animado

1. Criar o **modelo no Blockbench**.
2. Criar o **Geo Model** correspondente.
3. Criar o **item JSON**.
4. Criar o **item display JSON**.
5. Criar a **classe do item**.
6. Criar e registrar o **renderer**.

> Os passos 1 e 2 já são abordados nas seções de Geo Models; aqui focaremos nos passos 3 a 6.

---

## A Classe do Item

Para criar um item animável com GeckoLib:

1. Implemente `GeoItem` na classe do item.
2. Crie um **AnimatableInstanceCache** via `GeckoLibUtil.createInstanceCache(this)`.
3. Sobrescreva `getAnimatableInstanceCache()` retornando o cache.
4. Sobrescreva `registerControllers()` e defina suas animações.

> Para animações acionadas pelo servidor ou sincronizadas entre clientes, utilize `GeoItem.registerSyncedAnimatable(this)`.

### Triggerable Animations

* GeckoLib permite **ativar animações remotamente pelo servidor**, simplificando animações sincronizadas.

### Perspective-Aware Animation Handling

* Para animações dependentes da perspectiva (1ª ou 3ª pessoa), sobrescreva `GeoItem#isPerspectiveAware()` retornando `true`.
* Use `state.getData(DataTickets.ITEM_RENDER_PERSPECTIVE)` no seu predicate para diferenciar perspectivas.

---

## O Renderer do Item

* Crie uma classe que estenda `GeoItemRenderer`, passando o **Geo Model** para o construtor.

### Exemplo — Renderer

```java
public class ExampleItemRenderer extends GeoItemRenderer<ExampleItem> {
    public ExampleItemRenderer() {
        super(new ExampleItemModel());
    }
}
```

* Para registrar o renderer, sobrescreva `createGeoRenderer` na classe do item:

```java
@Override
public void createGeoRenderer(Consumer<GeoRenderProvider> consumer) {
    consumer.accept(new GeoRenderProvider() {
        private ExampleItemRenderer renderer;

        @Override
        public BlockEntityWithoutLevelRenderer getGeoItemRenderer() {
            if (this.renderer == null)
                this.renderer = new ExampleItemRenderer();

            return this.renderer;
        }
    });
}
```

* Alternativamente, você pode usar os métodos de registro nativos de Forge/Fabric, mas a compatibilidade futura não é garantida para todas as funcionalidades.

---

## O Item JSON

* Local: `assets/<modid>/items/<item_id>.json`
* Exemplo de conteúdo:

```json
{
  "model": {
    "type": "minecraft:special",
    "base": "<modid>:item/<item_id>",
    "model": {
      "type": "geckolib:geckolib"
    }
  }
}
```

> `base` deve apontar para o modelo do item em `/models/item/`.

---

## Exemplo de Classe Item

```java
public final class ExampleItem extends Item implements GeoItem {
    private static final RawAnimation ACTIVATE_ANIM = RawAnimation.begin().thenPlay("use.activate");
    private final AnimatableInstanceCache cache = GeckoLibUtil.createInstanceCache(this);

    public ExampleItem(Properties properties) {
        super(properties);
        GeoItem.registerSyncedAnimatable(this);
    }

    @Override
    public void createGeoRenderer(Consumer<GeoRenderProvider> consumer) {
        consumer.accept(new GeoRenderProvider() {
            private ExampleItemRenderer renderer;

            @Override
            public BlockEntityWithoutLevelRenderer getGeoItemRenderer() {
                if (this.renderer == null)
                    this.renderer = new ExampleItemRenderer();
                return this.renderer;
            }
        });
    }

    @Override
    public void registerControllers(AnimatableManager.ControllerRegistrar controllers) {
        controllers.add(new AnimationController<>("Activation", 0, animTest -> PlayState.STOP)
                .triggerableAnim("activate", ACTIVATE_ANIM));
    }

    @Override
    public InteractionResultHolder<ItemStack> use(Level level, Player player, InteractionHand hand) {
        if (level instanceof ServerLevel serverLevel)
            triggerAnim(player, GeoItem.getOrAssignId(player.getItemInHand(hand), serverLevel), "Activation", "activate");

        return super.use(level, player, hand);
    }

    @Override
    public AnimatableInstanceCache getAnimatableInstanceCache() {
        return this.cache;
    }
}
```

---

## Problemas Comuns

* **Item aparece como quadrado preto e roxo:** esqueceu de colocar o item JSON em `assets/<modid>/items`.
* **Não consigo repetir a animação:** lembre-se de resetar a animação ou evitar `hold_on_last_frame` no `.animation.json`.

# 🛡️ Armaduras Animadas com GeckoLib

> **Observação:** Ao criar armaduras no Blockbench, selecione **Armor model type** no painel Geckolib Model Settings. Isso garante que os arquivos de projeto e saída sejam gerados corretamente.

---

## Passos para criar uma armadura animada

1. Criar o **modelo no Blockbench**.
2. Criar o **Geo Model** correspondente.
3. Criar o **item JSON**.
4. Criar o **item display JSON**.
5. Criar a **classe da armadura**.
6. Criar e registrar o **renderer**.

> Os passos 1 e 2 são abordados nas seções de Geo Models, 3 e 4 na página de Itens. Aqui vamos focar nos passos 5 e 6.

---

## A Classe da Armadura

* Crie a classe estendendo `ArmorItem` para lidar automaticamente com o equipamento.
* Implemente `GeoItem` e siga a mesma lógica de itens animáveis.
* Crie `AnimatableInstanceCache` e sobrescreva `getAnimatableInstanceCache()`.
* Registre controladores de animação no `registerControllers()`.

### Exemplo — Classe de Armadura

```java
public final class ExampleArmorItem extends Item implements GeoItem {
    public static final DataTicket<Boolean> HAS_FULL_SET_EFFECT = DataTicket.create("examplemod_has_full_set_effect", Boolean.class);
    private final AnimatableInstanceCache cache = GeckoLibUtil.createInstanceCache(this);

    public ExampleArmorItem(ArmorMaterial armorMaterial, ArmorType type, Properties properties) {
        super(properties.humanoidArmor(armorMaterial, type));
    }

    @Override
    public void createGeoRenderer(Consumer<GeoRenderProvider> consumer) {
        consumer.accept(new GeoRenderProvider() {
            private GeoArmorRenderer<?> renderer;

            @Override
            public <T extends LivingEntity> HumanoidModel<?> getGeoArmorRenderer(@Nullable T livingEntity, ItemStack itemStack, @Nullable EquipmentSlot equipmentSlot, @Nullable HumanoidModel<T> original) {
                if(this.renderer == null)
                    this.renderer = new ExampleArmorRenderer();
                return this.renderer;
            }
        });
    }

    @Override
    public void registerControllers(AnimatableManager.ControllerRegistrar controllers) {
        controllers.add(new AnimationController<>(this, 20, animTest -> {
            if (animTest.getData(HAS_FULL_SET_EFFECT))
                return animTest.setAndContinue(DefaultAnimations.IDLE);
            return PlayState.STOP;
        }));
    }

    @Override
    public AnimatableInstanceCache getAnimatableInstanceCache() {
        return this.cache;
    }
}
```

---

## O Renderer da Armadura

* Estenda `GeoArmorRenderer` e forneça um `GeoModel` apropriado.
* O modelo padrão assume um humanoide com braços, pernas, torso e cabeça.
* Utilize os nomes das bones do `.geo.json` para mapear corretamente as partes da armadura.

### Exemplo — Renderer

```java
public final class ExampleArmorRenderer extends GeoArmorRenderer<ExampleArmorItem> {
    public ExampleArmorRenderer() {
        super(new DefaultedItemGeoModel<>(new ResourceLocation(ExampleMod.MOD_ID, "armor/example_armor")));
    }

    @Override
    public void addRenderData(ExampleArmorItem animatable, RenderData relatedObject, R renderState) {
        Set<Item> wornArmor = new ObjectOpenHashSet<>(4);
        boolean fullSetEffect = false;

        if (!(relatedObject.entity() instanceof ArmorStand)) {
            for (EquipmentSlot slot : EquipmentSlot.values()) {
                if (slot.getType() == EquipmentSlot.Type.HUMANOID_ARMOR)
                    wornArmor.add(relatedObject.entity().getItemBySlot(slot).getItem());
            }
            fullSetEffect = wornArmor.containsAll(ObjectArrayList.of(
                ItemRegistry.EXAMPLE_ARMOR_BOOTS.get(),
                ItemRegistry.EXAMPLE_ARMOR_LEGGINGS.get(),
                ItemRegistry.EXAMPLE_ARMOR_CHESTPLATE.get(),
                ItemRegistry.EXAMPLE_ARMOR_HELMET.get()));
        }

        renderState.addGeckolibData(ExampleArmorItem.HAS_FULL_SET_EFFECT, fullSetEffect);
    }
}
```

---

## Registrando o Renderer

* Sobrescreva `createGeoRenderer` na classe da armadura.
* Retorne uma instância do renderer usando `GeoRenderProvider`.
* Isso permite manipulação dinâmica e compatível do renderer.

### Exemplo — Registro do Renderer

```java
@Override
public void createGeoRenderer(Consumer<GeoRenderProvider> consumer) {
    consumer.accept(new GeoRenderProvider() {
        private ExampleArmorRenderer<?> renderer;

        @Nullable
        @Override
        public <S extends HumanoidRenderState> ExampleArmorRenderer<?> getGeoArmorRenderer(@Nullable S renderState, ItemStack itemStack, EquipmentSlot equipmentSlot,
                                                                                          EquipmentClientInfo.LayerType type, @Nullable HumanoidModel<S> original) {
            if(this.renderer == null)
                this.renderer = new ExampleArmorRenderer<>();
            return this.renderer;
        }
    });
}
```

# 🎨 RenderStates com GeckoLib 5

> **Visão Geral:** RenderStates são a nova abordagem da Mojang para manipulação de renderização, oferecendo maior compatibilidade com multithreading ao coletar dados imutáveis de objetos renderizados.

---

## Como isso afeta o GeckoLib?

* GeckoLib é uma biblioteca de renderização e precisa se adaptar às mudanças do Minecraft.
* GeckoLib 5.0 implementa compatibilidade com o novo pipeline de RenderStates.

## Por que GeckoLib aplica RenderStates a tudo?

* Mojang aplica apenas a Entity RenderStates atualmente.
* GeckoLib aplica a todos os renderers para manter consistência futura e evitar lidar com dois sistemas distintos.

## Funcionamento

* GeckoLib utiliza `GeoRenderState` para armazenar todos os dados de renderização.
* É criado antes de cada render frame.
* Dados são mapeados em `DataTickets` para uso no render.

### Métodos importantes do GeoRenderState

#### `#addGeckolibData`

* Adiciona dados ao RenderState para uso posterior.
* Dados repetidos sobrescrevem os anteriores.

#### `#getGeckolibData`

* Recupera dados previamente armazenados.
* Acesso a dados não existentes causa crash.

#### `#hasGeckolibData`

* Verifica se dados foram armazenados, mesmo que nulos.
* Deve ser usado antes de `#getGeckolibData` se não tiver certeza dos dados.

#### `#getOrDefaultGeckolibData`

* Retorna dados armazenados ou valor padrão caso não existam.

## Manipulação Padrão

* `#fillRenderState`: coleta dados base antes da renderização.
* `#captureDefaultRenderState`: tratamento interno do GeckoLib.
* `#addRenderData`: método para adicionar/modificar dados no GeoRenderState.
* `GeoRenderLayer#addRenderData`: adiciona dados específicos para camadas de render.
* `GeoModel#prepareForRenderPass`: adiciona/modifica dados específicos do modelo.

> Observação: Alguns renderers, como `GeoEntityRenderer`, possuem tipos covariantes combinando `EntityRenderState` com `GeoRenderState`.

## Uso

### 1️⃣ Usando DataTickets (recomendado)

* Armazena e recupera dados via `DataTickets`.
* Permite interação entre mods sem dependência direta.

### 2️⃣ Estendendo GeoRenderState

* Crie uma classe que estenda `GeoRenderState` ou seu tipo covariante.
* Permite uso de campos de instância em vez de DataTickets.
* Desvantagens:

  * Mods dependentes precisam classloadar sua classe.
  * Pode exigir tipo covariante genérico no renderer.

> Benefício: melhor desempenho se o renderer acessa dados intensamente.

## Tipos Covariantes

* Permite que o renderer trate o objeto como dois tipos simultaneamente.

### Exemplo de Covariância

```java
public class GeoEntityRenderer<T extends Entity & GeoAnimatable, R extends EntityRenderState & GeoRenderState>
    extends EntityRenderer<T, R> implements GeoRenderer<T, Void, R> {}
```

* `T`: Entity & GeoAnimatable

* `R`: EntityRenderState & GeoRenderState

* Ao estender renderers com tipos covariantes, mantenha a covariância nos genéricos.

### Exemplo de Extensão

```java
public class ExampleEntityRenderer<R extends ExampleRenderState & GeoRenderState>
    extends GeoEntityRenderer<ExampleEntity, R> {}
```

# 🎬 Definindo Animações em Código (Geckolib 5)

> **Objetivo:** Explorar nuances da criação e controle de animações via código no GeckoLib 5, incluindo controllers, transições e easings.

---

## Fundamentos de Animação

### GeoAnimatable

* Todo objeto animável deve implementar **`GeoAnimatable`** ou suas variações (`GeoItem`, `GeoBlockEntity`, etc.).
* Requer sobrescrever:

  * `getAnimatableInstanceCache`
  * `registerControllers`

### AnimatableInstanceCache

* Armazena instâncias animáveis para cada objeto.
* Garante que instâncias distintas não compartilhem animação simultaneamente.
* Sempre instanciar via `GeckoLibUtil.createInstanceCache(this)`.

### AnimationController

* Cada controller gerencia **uma animação por vez**.
* Para múltiplas animações concorrentes, registre **múltiplos controllers**.
* Conflitos podem ocorrer se duas animações afetarem o mesmo osso.
* Controllers são executados na ordem de registro; menor índice → maior prioridade.

### Estados do Controller

| Estado        | Significado                                              |
| ------------- | -------------------------------------------------------- |
| Running       | Animação está sendo executada                            |
| Transitioning | Mudança de uma animação para outra                       |
| Stopped       | Controller parado ou ossos retornando ao estado original |

## Manipulação de Animações

* Inicie/execute uma animação com `AnimationController.setAnimation`.
* Animações não-loop devem ser resetadas com `AnimationController.forceAnimationReset` para serem reproduzidas novamente.

### AnimationTest

* Função chamada a cada render pass.
* Retorno: `PlayState.CONTINUE` para continuar a animação, `PlayState.STOP` para interromper.

## Transições de Animações e Ossos

* Por padrão, a transição é **linear**.
* Controlada pelo campo `transitionLength` do controller.

  * 0 → transição instantânea.
* Para reset de ossos após animação, sobrescreva `#getBoneResetTime` no `GeoAnimatable`.

## Easings

* Determina a interpolação entre posições de ossos.
* Pode ser configurado via JSON ou via `AnimationController.setOverrideEasingType`.
* Valor `null` → retorna ao easing do JSON.

### Custom Easing

* Crie easings personalizados no startup do mod:

```java
GeckoLibUtil.addCustomEasingType("nomeEasing", input -> {
    // input: valor de 0 a 1
    return processedValue;
});
```

* A função recebe um `double` de 0 a 1 e retorna outro `double` correspondente à posição interpolada.
* Referência: [easings.net](https://easings.net)

---

*Capítulo criado com base na seção "Defining Animations in Code" da wiki do GeckoLib 5.*

# AprendeIA — Render Layers (Geckolib 5)

> **Documento de aprendizagem gerado pelo "gerrador" (IA).**

---

## Objetivo

Apresentar o funcionamento das **Render Layers** no Geckolib 5, permitindo adicionar camadas de renderização adicionais sobre o render padrão de entidades, blocos, items ou armaduras.

---

# 🌐 Conceito de Render Layers

Render Layers no Geckolib 5 são representadas por instâncias de **GeoRenderLayer**. Elas permitem que você desenhe conteúdo adicional sobre o render principal de um animável.

### Uso Básico

Ao instanciar o renderizador, adicione uma nova camada com o método `addRenderLayer`, passando a instância do seu GeoRenderLayer.

```java
public class ExampleRenderer<R extends LivingEntityRenderState & GeoRenderState>
        extends GeoEntityRenderer<ExampleEntity, R> {
    public ExampleRenderer(EntityRendererProvider.Context renderManager) {
        super(renderManager, new ExampleEntityModel());
        
        addRenderLayer(new ExampleGeoRenderLayer(this));
    }
}
```

**Atenção:**

* Se você tentar re-renderizar o mesmo modelo dentro do método `#render` da sua GeoLayer, chame `#reRender` para informar ao Geckolib que é apenas uma renderização de camada.

---

# 🛠 Built-In Render Layers

O Geckolib fornece várias camadas prontas para uso.

### AutoGlowingGeoLayer

* Renderiza uma camada emissiva/glow/fullbright.
* **Uso:** `addRenderLayer(new AutoGlowingGeoLayer<>(this));`

### BlockAndItemGeoLayer

* Renderiza BlockStates ou ItemStacks em uma posição de bone específica.
* Requer uma lista de bones e RenderData.

### CustomBoneTextureGeoLayer

* Similar ao antigo Dynamic Renderer do Geckolib 4.
* Re-dimensiona automaticamente a textura para o bone.
* **Uso:**

```java
addRenderLayer(new CustomBoneTextureGeoLayer(this,
               "dynamic_bone",
               ResourceLocation.fromNamespaceAndPath(ExampleMod.MOD_ID, "textures/entity/example_entity_custom_bone.png")));
```

* Pode sobrescrever `#getRenderType` para modificar o tipo de renderização.

### ItemArmorGeoLayer

* Renderiza armaduras sobre a GeoEntity.
* Requer lista de RenderData definindo peças, slot e parte do modelo.
* **Observação:** o primeiro cube do GeoBone define posicionamento/rotação/escala.
* **Uso:**

```java
addRenderLayer(new ItemArmorGeoLayer<>(this, context) {
    private final List<RenderData> BONES = List.of(
        RenderData.head(HELMET_BONE), RenderData.body(CHESTPLATE_BONE),
        RenderData.leftArm(LEFT_SLEEVE_BONE), RenderData.rightArm(RIGHT_SLEEVE_BONE),
        RenderData.leftLeg(LEFT_ARMOR_LEG_BONE), RenderData.rightLeg(RIGHT_ARMOR_LEG_BONE),
        RenderData.leftFoot(LEFT_BOOT_BONE), RenderData.rightFoot(RIGHT_BOOT_BONE)
    );

    @Override
    protected List<RenderData> getRelevantBones(R renderState, BakedGeoModel model) {
        return BONES;
    }
});
```

### ItemInHandGeoLayer

* Renderiza items nas mãos da GeoEntity.
* Requer apenas um bone invisível para a posição de cada mão (`RightHandItem` e `LeftHandItem`).
* **Uso:** `addRenderLayer(new ItemInHandGeoLayer(this));`

### TextureLayerGeoLayer

* Renderiza uma textura adicional sobre o animável.
* Útil para skins secundárias ou overlays.
* **Uso:**

```java
addRenderLayer(new TextureLayerGeoLayer<>(this,
               ResourceLocation.fromNamespaceAndPath(ExampleMod.MOD_ID, "textures/entity/glasses.png"),
               RenderType::armorCutoutNoCull));
```

---

*Documento atualizado com base na página "Render Layers (Geckolib 5)" da wiki oficial.*


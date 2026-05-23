# Code Review — Portfolio Angular

Olá! Fiz uma análise detalhada do projeto e deixei abaixo observações organizadas por arquivo, cobrindo code smells, boas práticas, sugestões de refatoração e padrões de projeto. O projeto tem uma base sólida e visual interessante — os comentários abaixo visam elevar ainda mais a qualidade do código.

---

## `translation.service.ts`

### 1. `ngOnInit` não funciona em `@Injectable`

**Problema:** A classe `TranslationService` implementa `OnInit` e define `ngOnInit()`, mas serviços Angular não possuem ciclo de vida `ngOnInit`. O método nunca será chamado pelo framework.

**Sugestão:** Remova a interface `OnInit` e o `ngOnInit`. Se precisar de inicialização, faça diretamente no `constructor`, como já é feito com o `loadTranslation`.

```typescript
//  Atual — ngOnInit nunca é chamado em serviços
export class TranslationService implements OnInit {
  ngOnInit() {
    console.log("ts service started");
  }
}

//  Correto — inicialização no constructor
export class TranslationService {
  constructor(private http: HttpClient) {
    this.loadTranslation(this.currentLang);
  }
}
```

---

### 2. `console.log` em produção

**Arquivo:** `translation.service.ts`

**Problema:** O método `loadTranslation` contém um `console.log(this.translation)` que expõe dados internos do serviço no console em ambiente de produção.

**Sugestão:** Remova logs de depuração antes de fazer deploy, ou utilize uma estratégia condicional baseada no `environment`.

```typescript
// Atual
this.translation = data;
console.log(this.translation);

//  Sugerido
this.translation = data;
if (!environment.production) {
  console.log(this.translation);
}
```

---

### 3. Ausência de tratamento de erros no `loadTranslation`

**Arquivo:** `translation.service.ts`

**Problema:** A chamada HTTP em `loadTranslation` não possui nenhum tratamento de erro. Se o arquivo de tradução não for encontrado (404) ou a rede falhar, o objeto `translation` permanecerá vazio e a interface exibirá as chaves brutas no lugar dos textos.

**Sugestão:** Adicione `catchError` com o operador do RxJS para lidar com falhas de forma elegante.

```typescript
import { catchError } from 'rxjs/operators';
import { of } from 'rxjs';

loadTranslation(lang: 'br' | 'en') {
  return this.http.get(`/i18n/${lang}.json`).pipe(
    catchError(err => {
      console.error('Falha ao carregar tradução:', err);
      return of({});
    })
  ).subscribe((data: any) => {
    this.translation = data;
    this.currentLang = lang;
  });
}
```

---

### 4. Ausência de reatividade — `translate()` não notifica a view

**Arquivo:** `translation.service.ts`

**Problema:** O método `translate(key)` retorna um valor estático. Quando o idioma é trocado via `loadTranslation`, a view não é notificada automaticamente para re-renderizar, pois não há nenhum mecanismo reativo (como `BehaviorSubject` ou `Signal`).

**Sugestão de padrão:** Utilize um `BehaviorSubject` para expor as traduções de forma reativa, permitindo que os componentes se inscrevam e reajam à mudança de idioma.

```typescript
private translationSubject = new BehaviorSubject<any>({});
translation$ = this.translationSubject.asObservable();

loadTranslation(lang: 'br' | 'en') {
  this.http.get(`/i18n/${lang}.json`).subscribe((data: any) => {
    this.translationSubject.next(data);
    this.currentLang = lang;
  });
}

translate(key: string): string {
  return this.translationSubject.getValue()[key] || key;
}
```

---

## `app.ts`

### 5. Variável `theme` global no escopo do módulo (anti-pattern)

**Arquivo:** `app.ts`

**Problema:** `theme` é declarada como variável `let` no escopo do módulo, fora da classe. Isso cria um estado global compartilhado que quebra o encapsulamento e pode causar bugs difíceis de rastrear, especialmente se o projeto crescer. Também impede que o Angular gerencie o estado corretamente via injeção de dependência.

```typescript
//  Atual — variável global no módulo
let theme: 'light' | 'dark' = 'dark';

@Component({ ... })
export class App { ... }

export function invokeParticles(): void {
  particlesJS('particles-js', theme === 'dark' ? ParticlesConfigDark : ParticlesConfigLight, ...);
}
```

**Sugestão:** Encapsule o tema dentro de um `ThemeService` e injete-o onde necessário. A função `invokeParticles` pode receber o tema como parâmetro.

```typescript
//  Sugerido
export function invokeParticles(theme: 'light' | 'dark'): void {
  particlesJS('particles-js', theme === 'dark' ? ParticlesConfigDark : ParticlesConfigLight, () => {});
}

// No componente:
toggleTheme() {
  this.theme = this.theme === 'dark' ? 'light' : 'dark';
  invokeParticles(this.theme);
}
```

---

### 6. `declare let particlesJS` — integração com biblioteca sem tipagem

**Arquivo:** `app.ts`

**Problema:** O uso de `declare let particlesJS: any` suprime completamente a verificação de tipos do TypeScript para a biblioteca `particles.js`. Qualquer erro de configuração passará despercebido em tempo de compilação.

**Sugestão:** Instale os tipos da biblioteca (se disponíveis) ou crie um arquivo de declaração de tipos (`particles.d.ts`) com uma tipagem mínima para os parâmetros utilizados.

---

### 7. Callback vazio desnecessário em `particlesJS`

**Arquivo:** `app.ts`

**Problema:** A função `invokeParticles` passa `function () {}` como terceiro argumento para `particlesJS`. Esse callback nunca é usado e polui o código sem propósito.

```typescript
//  Atual
particlesJS("particles-js", config, function () {});

//  Sugerido — omitir ou usar arrow function vazia de forma explícita
particlesJS("particles-js", config);
```

---

## `app.spec.ts`

### 8. Teste desatualizado — verifica título que não existe no template

**Arquivo:** `app.spec.ts`

**Problema:** O teste `'should render title'` verifica se existe um `<h1>` com o texto `"Hello, PortfolioLaboratorio"` no template, mas o template real (`app.html`) não contém esse conteúdo — ele foi gerado automaticamente pelo Angular CLI e nunca atualizado.

**Sugestão:** Atualize o spec para refletir o comportamento real da aplicação, ou remova o teste obsoleto. Testes que verificam comportamentos inexistentes geram falsa confiança.

```typescript
//  Atual — testa algo que não existe no template real
expect(compiled.querySelector("h1")?.textContent).toContain(
  "Hello, PortfolioLaboratorio",
);

//  Sugerido — teste relevante para a aplicação real
it("should contain the scroll container", () => {
  fixture.detectChanges();
  const compiled = fixture.nativeElement as HTMLElement;
  expect(compiled.querySelector('[class*="h-screen"]')).toBeTruthy();
});
```

---

## `experiencia.components.ts`

### 9. Dados hardcoded no componente — violação do Single Responsibility Principle

**Arquivo:** `experiencia.components.ts`

**Problema:** O array `experiencias` com todos os dados está definido diretamente dentro do componente. Isso viola o **Single Responsibility Principle (SRP)**: o componente deveria apenas exibir dados, não os armazenar. Além disso, dificulta manutenção, testes e futura integração com uma API.

**Sugestão de padrão (Repository/Service Pattern):** Extraia os dados para um serviço `ExperienciaService`. Isso desacopla a camada de dados da camada de apresentação.

```typescript
// experiencia.service.ts
@Injectable({ providedIn: "root" })
export class ExperienciaService {
  getExperiencias(): Experiencia[] {
    return [
      /* dados aqui */
    ];
  }
}

// experiencia.component.ts
export class ExperienciaComponent {
  experiencias = inject(ExperienciaService).getExperiencias();
}
```

---

### 10. Interface `Experiencia` definida no mesmo arquivo do componente

**Arquivo:** `experiencia.components.ts`

**Problema:** A interface `Experiencia` está declarada junto ao componente. Em projetos Angular organizados, interfaces e tipos são colocados em arquivos separados (ex.: `experiencia.model.ts` ou `models/experiencia.interface.ts`).

**Sugestão:** Crie um arquivo dedicado para os modelos:

```
src/app/models/experiencia.interface.ts
```

Isso melhora a reusabilidade e facilita a importação em outros contextos (como um serviço ou outro componente).

---

### 11. URLs corrompidas nas imagens das experiências

**Arquivo:** `experiencia.components.ts`

**Problema:** Dois itens do array `experiencias` contêm URLs de imagem que apontam para resultados de busca do Google (`google.com/imgres?...`) em vez de imagens diretas. Essas URLs nunca renderizarão uma imagem corretamente.

```typescript
//  URL inválida para imagem
image: "https://www.google.com/imgres?q=redhat&imgurl=...";
```

**Sugestão:** Utilize a URL direta da imagem (o valor do parâmetro `imgurl`) ou hospede as imagens no próprio projeto em `assets/images/`.

---

### 12. Dados de placeholder em produção

**Arquivo:** `experiencia.components.ts`

**Problema:** Os dois últimos itens do array `experiencias` possuem dados claramente provisórios (`'Título da Nova Experiência'`, `'2026'`, `via.placeholder.com`). Conteúdo de placeholder não deve estar presente em commits de produção.

**Sugestão:** Remova os itens fictícios ou mova-os para um branch separado de desenvolvimento.

---

### 13. Nome do arquivo com nomenclatura inconsistente

**Arquivo:** `experiencia.components.ts` (no lugar de `experiencia.component.ts`)

**Problema:** O arquivo está nomeado `experiencia.components.ts` (plural), enquanto a convenção do Angular é sempre usar o singular `*.component.ts`. Isso pode causar confusão e quebrar a consistência do projeto.

**Sugestão:** Renomeie para `experiencia.component.ts` seguindo o padrão oficial do Angular Style Guide.

---

## `experiencia.html`

### 14. Manipulação do DOM com `onclick` inline — quebra o padrão Angular

**Arquivo:** `experiencia.html`

**Problema:** Os botões de navegação do carrossel usam `onclick` inline com `document.getElementById`, o que é uma prática jQuery-style incompatível com a filosofia declarativa do Angular. Isso também cria dependência direta do DOM e impede testes unitários.

```html
<!--  Atual -->
<button
  onclick="document.getElementById('carousel').scrollBy({ left: -400, behavior: 'smooth' })"
></button>
```

**Sugestão:** Utilize `@ViewChild` no componente para obter a referência ao elemento e crie métodos dedicados:

```typescript
@ViewChild('carousel') carousel!: ElementRef<HTMLDivElement>;

scrollLeft() { this.carousel.nativeElement.scrollBy({ left: -400, behavior: 'smooth' }); }
scrollRight() { this.carousel.nativeElement.scrollBy({ left: 400, behavior: 'smooth' }); }
```

```html
<!--  Sugerido -->
<button (click)="scrollLeft()">&#10094;</button>
<div #carousel class="carousel">...</div>
<button (click)="scrollRight()">&#10095;</button>
```

---

### 15. `track` com valor que pode não ser único

**Arquivo:** `experiencia.html`

**Problema:** O `@for` usa `track exp.titleKey`, mas `titleKey` armazena a string do título diretamente (ex.: `'Técnico em Informática'`). Se dois itens tiverem o mesmo título, o Angular não conseguirá diferenciar os nós do DOM, causando comportamento inesperado na renderização.

**Sugestão:** Adicione um campo `id` único à interface `Experiencia` e use-o no `track`:

```typescript
export interface Experiencia {
  id: number;
  // ...demais campos
}
```

```html
@for (exp of experiencias; track exp.id) { ... }
```

---

## `sobre.html`

### 16. URL de imagem externa com token de autenticação

**Arquivo:** `sobre.html`

**Problema:** A tag `<img>` aponta para uma URL do Instagram com tokens de autenticação e parâmetros de sessão (`oh=`, `oe=`, `_nc_gid=`, etc.). Esses tokens expiram periodicamente, fazendo com que a imagem deixe de carregar sem nenhum aviso.

**Sugestão:** Baixe a imagem e hospede-a localmente em `src/assets/images/foto-perfil.jpg`. Isso garante disponibilidade permanente e elimina a dependência de uma CDN de terceiros.

```html
<!--  Sugerido -->
<img
  src="assets/images/foto-perfil.jpg"
  alt="Foto de perfil de Nicolas Araújo"
/>
```

---

### 17. Atributo `alt` vazio na imagem de perfil

**Arquivo:** `sobre.html`

**Problema:** O elemento `<img>` da foto de perfil possui `alt=""`. Imagens informativas devem ter um texto alternativo descritivo para garantir acessibilidade (WCAG 2.1) e melhorar o SEO.

```html
<!--  Atual -->
<img ... alt="" />

<!--  Sugerido -->
<img ... alt="Foto de perfil de Nicolas Araújo, Engenheiro de Software" />
```

---

Espero que os comentários sejam úteis! O projeto tem uma identidade visual bacana e uma estrutura Angular bem organizada no geral. Com os ajustes acima, ele ficará mais robusto, manutenível e alinhado com as boas práticas da plataforma. 

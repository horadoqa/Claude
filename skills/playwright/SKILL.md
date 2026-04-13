---
name: playwright-serverest
description: >
  Use este skill para criar, executar e manter testes E2E com Playwright
  no site https://front.serverest.dev e na API https://serverest.dev.
  Acione sempre que o usuário mencionar: testes de cadastro, login, Playwright,
  ServeRest, automação E2E, testes de usuário, front.serverest, ou qualquer
  combinação de Playwright com ServeRest. Mesmo que o usuário não mencione
  "skill", use-o sempre que o contexto envolver automação de testes neste projeto.
---

# Playwright — Testes de cadastro no ServeRest

Skill para criação e execução de testes E2E com Playwright no ServeRest.

- **Front-end:** https://front.serverest.dev
- **API (Back-end):** https://serverest.dev
- **Documentação da API:** https://serverest.dev/#/

---

## Contexto da aplicação

O ServeRest simula uma loja virtual com dois perfis:

| Perfil        | Acesso após cadastro/login          |
|---------------|-------------------------------------|
| Administrador | Dashboard admin, CRUD de usuários e produtos |
| Cliente       | Home com produtos, carrinho         |

### Fluxo de cadastro (front-end)

1. Acessar `https://front.serverest.dev`
2. Clicar em **"Cadastre-se"** na tela de login
3. Preencher: Nome, E-mail, Password
4. Marcar ou não o checkbox **"Cadastrar como administrador"**
5. Clicar em **"Cadastrar"**
6. Redirecionamento automático:
   - Admin → `/admin/home`
   - Cliente → `/home`

### Seletores dos elementos

| Elemento                  | Seletor recomendado                        |
|---------------------------|--------------------------------------------|
| Campo Nome                | `input[data-testid="nome"]`                |
| Campo E-mail              | `input[data-testid="email"]`               |
| Campo Senha               | `input[data-testid="password"]`            |
| Checkbox Administrador    | `input[data-testid="checkbox"]`            |
| Botão Cadastrar           | `button[data-testid="cadastrar"]`          |
| Link "Cadastre-se"        | `a[data-testid="cadastrar"]` (na tela login) |
| Mensagem de alerta/erro   | `.alert` ou `[role="alert"]`               |

---

## Estrutura do projeto

```
playwright-serverest/
├── SKILL.md
├── playwright.config.js
├── package.json
├── tests/
│   └── e2e/
│       └── cadastro.e2e.test.js   ← testes principais
├── support/
│   └── e2e/
│       └── CadastroPage.js        ← Page Object
└── references/
    └── cenarios.md                ← todos os cenários de teste
```

---

## Setup do projeto

### 1. Instalar dependências

```bash
npm init -y
npm install -D @playwright/test @faker-js/faker
npx playwright install chromium
```

### 2. playwright.config.js

```javascript
const { defineConfig } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './tests',
  timeout: 30000,
  retries: 1,
  use: {
    baseURL: 'https://front.serverest.dev',
    headless: true,
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  reporter: [['html', { open: 'never' }]],
});
```

### 3. package.json (scripts)

```json
{
  "scripts": {
    "test:e2e": "npx playwright test tests/e2e",
    "test:headed": "npx playwright test tests/e2e --headed",
    "report": "npx playwright show-report"
  }
}
```

---

## Page Object — CadastroPage.js

Sempre use Page Objects para encapsular os seletores e ações da página.

```javascript
// support/e2e/CadastroPage.js
class CadastroPage {
  constructor(page) {
    this.page = page;
    this.nomeInput     = page.locator('input[data-testid="nome"]');
    this.emailInput    = page.locator('input[data-testid="email"]');
    this.passwordInput = page.locator('input[data-testid="password"]');
    this.adminCheckbox = page.locator('input[data-testid="checkbox"]');
    this.cadastrarBtn  = page.locator('button[data-testid="cadastrar"]');
    this.alerta        = page.locator('.alert');
  }

  async navegarParaCadastro() {
    await this.page.goto('/');
    await this.page.locator('a[data-testid="cadastrar"]').click();
  }

  async preencherFormulario({ nome, email, password, isAdmin = false }) {
    await this.nomeInput.fill(nome);
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    if (isAdmin) {
      await this.adminCheckbox.check();
    }
  }

  async submeter() {
    await this.cadastrarBtn.click();
  }

  async cadastrar(dados) {
    await this.preencherFormulario(dados);
    await this.submeter();
  }
}

module.exports = { CadastroPage };
```

---

## Testes — cadastro.e2e.test.js

Consulte `references/cenarios.md` para a lista completa de cenários.
Abaixo estão os cenários mais importantes implementados:

```javascript
// tests/e2e/cadastro.e2e.test.js
const { test, expect } = require('@playwright/test');
const { faker } = require('@faker-js/faker/locale/pt_BR');
const { CadastroPage } = require('../../support/e2e/CadastroPage');

test.describe('Cadastro de usuário — ServeRest', () => {

  test('deve cadastrar usuário comum com sucesso', async ({ page }) => {
    const cadastro = new CadastroPage(page);
    await cadastro.navegarParaCadastro();

    await cadastro.cadastrar({
      nome:     faker.person.fullName(),
      email:    faker.internet.email(),
      password: faker.internet.password({ length: 8 }),
      isAdmin:  false,
    });

    await expect(page).toHaveURL(/\/home/);
  });

  test('deve cadastrar usuário administrador com sucesso', async ({ page }) => {
    const cadastro = new CadastroPage(page);
    await cadastro.navegarParaCadastro();

    await cadastro.cadastrar({
      nome:     faker.person.fullName(),
      email:    faker.internet.email(),
      password: faker.internet.password({ length: 8 }),
      isAdmin:  true,
    });

    await expect(page).toHaveURL(/\/admin\/home/);
  });

  test('deve exibir erro ao tentar cadastrar com e-mail já existente', async ({ page }) => {
    const cadastro = new CadastroPage(page);
    const emailFixo = 'usuario.existente@qa.com';

    // Primeiro cadastro
    await cadastro.navegarParaCadastro();
    await cadastro.cadastrar({
      nome:     'Usuário Existente',
      email:    emailFixo,
      password: 'senha123',
    });

    // Segundo cadastro com mesmo e-mail
    await cadastro.navegarParaCadastro();
    await cadastro.cadastrar({
      nome:     'Outro Nome',
      email:    emailFixo,
      password: 'senha456',
    });

    await expect(cadastro.alerta).toBeVisible();
    await expect(cadastro.alerta).toContainText('Este email já está sendo usado');
  });

  test('deve exibir erro ao submeter formulário vazio', async ({ page }) => {
    const cadastro = new CadastroPage(page);
    await cadastro.navegarParaCadastro();
    await cadastro.submeter();

    await expect(cadastro.alerta).toBeVisible();
  });

  test('deve exibir erro com e-mail em formato inválido', async ({ page }) => {
    const cadastro = new CadastroPage(page);
    await cadastro.navegarParaCadastro();

    await cadastro.cadastrar({
      nome:     faker.person.fullName(),
      email:    'email-invalido',
      password: faker.internet.password({ length: 8 }),
    });

    await expect(cadastro.alerta).toBeVisible();
  });

});
```

---

## Executando os testes

```bash
# Rodar todos os testes E2E
npm run test:e2e

# Rodar com browser visível (útil para debug)
npm run test:headed

# Ver relatório HTML
npm run report

# Rodar um teste específico
npx playwright test -g "deve cadastrar usuário comum"

# Rodar em modo debug (passo a passo)
npx playwright test --debug
```

---

## Boas práticas

- Use `faker` para gerar dados dinâmicos e evitar conflitos de e-mail duplicado
- Sempre prefira seletores `data-testid` — são mais estáveis que CSS ou XPath
- Use Page Objects para isolar seletores dos testes
- Configure `screenshot` e `video` como `retain-on-failure` para facilitar debug
- Evite `page.waitForTimeout()` — prefira `expect(...).toBeVisible()` ou `toHaveURL()`

---

## Referências

- Cenários completos de teste → `references/cenarios.md`
- Documentação Playwright → https://playwright.dev/docs/intro
- API ServeRest → https://serverest.dev/#/
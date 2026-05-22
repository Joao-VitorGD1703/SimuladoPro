# SimuladoPro

O seu ambiente de testes rápido e eficiente. Crie simulados dinâmicos e instantâneos colando ou enviando um simples arquivo JSON.

## 🚀 Acesso ao App (Online)

Você pode testar e utilizar o aplicativo diretamente no navegador através do link abaixo:

🔗 **[Acessar SimuladoPro](https://simulado-4sg8rb8s1-joaovitorgd1703s-projects.vercel.app/)**

---

## 📖 Como usar

1. Entre no site através do link acima.
2. Na aba **Colar JSON**, insira as suas perguntas no formato estruturado, ou use a aba **Fazer Upload** para arrastar um arquivo físico `.json` da sua máquina.
3. O simulado será gerado instantaneamente e você já pode começar a responder. Suas respostas são salvas e avaliadas ao final!

## 🧩 Formato do JSON Suportado

O arquivo JSON (ou o texto colado) deve obrigatoriamente seguir a estrutura de um array de objetos, onde cada objeto representa uma pergunta. Veja o exemplo:

```json
[
  {
    "question": "Qual é a capital do Brasil?",
    "options": ["Rio de Janeiro", "São Paulo", "Brasília", "Salvador"],
    "answer": "Brasília"
  },
  {
    "question": "Quanto é 2 + 2?",
    "options": ["3", "4", "5", "6"],
    "answer": "4"
  }
]
```

## 💻 Desenvolvimento Local

Se você quiser rodar o projeto na sua máquina:

1. Clone este repositório.
2. Instale as dependências com `npm install`.
3. Rode o servidor de desenvolvimento com `npm run dev`.

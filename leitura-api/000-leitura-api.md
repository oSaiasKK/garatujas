// Define o caminho do arquivo JSON onde os dados serão armazenados
const jsonFilePath = __dirname + '/data.temp.json';
// Array que armazena os itens em memória
const list: string[] = await loadFromFile();


//Carrega os itens do arquivo JSON
//Se o arquivo não existir, retorna um array vazio
//Se houver outro erro, lança a exceção
async function loadFromFile() {
  try {
    // Lê o arquivo JSON usando a API do Bun
    const file = Bun.file(jsonFilePath);
    // Converte o conteúdo para texto
    const content = await file.text();
    // Faz parse do JSON e garante que é um array de strings
    return JSON.parse(content) as string[];
  } catch (error: any) {
    // Se o arquivo não existe (ENOENT = Error NO ENTry), retorna array vazio
    if (error.code === 'ENOENT')
      return [];
    // Qualquer outro erro é relançado
    throw error;
  }
}


//Salva o array 'list' no arquivo JSON
//Sobrescreve o arquivo com os dados atualizados
async function saveToFile() {
  try {
    // Escreve o array convertido para JSON no arquivo
    await Bun.write(jsonFilePath, JSON.stringify(list));
  } catch (error: any) {
    // Lança um erro customizado com mensagem em português
    throw new Error("Erro ao salvar os dados no arquivo: " + error.message);
  }
}


//Adiciona um novo item ao final do array
//Salva automaticamente no arquivo após adicionar
async function addItem(item: string) {
  list.push(item);
  await saveToFile();
}


//Retorna todos os itens do array
async function getItems() {
  return list;
}



//Atualiza um item em um índice específico
//Valida se o índice está dentro dos limites do array
async function updateItem(index: number, newItem: string) {
  // Verifica se o índice é válido
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");
  // Substitui o item no índice especificado
  list[index] = newItem;
  // Salva as alterações no arquivo
  await saveToFile();
}



//Remove um item do array em um índice específico
//Valida se o índice está dentro dos limites
async function removeItem(index: number) {
  // Verifica se o índice é válido
  if (index < 0 || index >= list.length)
    throw new Error("Index fora dos limites");
  // Remove 1 elemento a partir do índice especificado
  list.splice(index, 1);
  // Salva as alterações no arquivo
  await saveToFile();
}


// Exporta todas as funções para uso em outros arquivos
export default { addItem, getItems, updateItem, removeItem };


// ==================== ARQUIVO: server.ts ====================

import todo from "./core.ts";

// Cria e inicia o servidor HTTP na porta 3000
const server = Bun.serve({
  port: 3000,

  // Define as rotas da API
  routes: {
    // Rota raiz - serve o arquivo HTML estático
    "/": new Response(Bun.file("./public/index.html")),

    // Rotas da API de TODOs
    "/api/todo": {
      // GET: Retorna lista de todos os itens
      GET: async () => {
        const items = await todo.getItems()
        return Response.json(items)
      },

      // POST: Adiciona um novo item
      POST: async (req) => {
        // Extrai os dados do corpo da requisição
        const data = await req.json() as any;
        // Obtém o item, ou null se não for fornecido
        const item = data.item || null;
        // Valida se o item foi fornecido
        if (!item)
          return Response.json('Por favor, forneça um item para adicionar.', { status: 400 });
        // Adiciona o item
        await todo.addItem(item);
        // Retorna os dados enviados como confirmação
        return Response.json(data);
      },
    },

    // Rota para operações em um item específico (por índice)
    "/api/todo/:index": {
      // PUT: Atualiza um item completo em um índice específico
      PUT: async (req) => {
        // Extrai e converte o índice do parâmetro da rota
        const index = parseInt(req.params.index);
        // Valida se o índice é um número inteiro válido
        if (isNaN(index))
          return Response.json('Índice inválido. Um número inteiro é esperado.', { status: 400 });
        // Extrai os dados da requisição
        const data = await req.json() as any;
        // Obtém o novo item
        const newItem = data.newItem || null;
        // Valida se o novo item foi fornecido
        if (!newItem)
          return Response.json('Por favor, forneça um novo item para atualizar.', { status: 400 });
        try {
          // Atualiza o item no índice especificado
          await todo.updateItem(index, newItem);
          // Retorna mensagem de sucesso
          return Response.json(`Item no índice ${index} atualizado para "${newItem}".`);
        } catch (error: any) {
          // Captura e retorna erros (como índice fora dos limites)
          return Response.json(error.message, { status: 400 });
        }
      },

      // DELETE: Remove um item de um índice específico
      DELETE: async (req) => {
        // Extrai e converte o índice do parâmetro da rota
        const index = parseInt(req.params.index);
        // Valida se o índice é um número válido
        if (isNaN(index))
          return Response.json('Índice inválido.', { status: 400 });
        try {
          // Remove o item no índice especificado
          await todo.removeItem(index);
          // Retorna mensagem de sucesso
          return Response.json(`Item no índice ${index} removido com sucesso.`);
        } catch (error: any) {
          // Captura e retorna erros
          return Response.json(error.message, { status: 400 });
        }
      },
    },

    // ==================== ROTAS DE EXEMPLO ====================

    "/api/exemplo": {
      // GET: Retorna um texto com o timestamp atual
      GET: () => {
        return new Response(`Esse é o exemplo: ${Date.now()}`)
      },

      // POST: Recebe dados e adiciona a data de recebimento
      POST: async (req) => {
        // Extrai os dados do corpo da requisição
        const data = await req.json() as any;
        // Adiciona a data atual no formato brasileiro
        data.recebidoEm = new Date().toLocaleDateString("pt-BR");
        // Retorna os dados com a data adicionada
        return Response.json(data);
      },
    },

    "/api/exemplo/:id": {
      // PUT: Atualiza recurso completo (substitui todos os campos)
      PUT: async (req, params) => {
        // Extrai o ID do parâmetro da rota
        const { id } = req.params;
        // Extrai os dados da requisição
        const data = await req.json() as any;
        // Adiciona o ID aos dados de resposta
        data.id = id;
        // Adiciona a data atual em formato brasileiro
        data.recebidoEm = new Date().toLocaleDateString("pt-BR");
        // Retorna os dados com ID e data
        return Response.json(data);
      },

      // PATCH: Atualização parcial (apenas campos fornecidos)
      PATCH: async (req, params) => {
        // Extrai o ID do parâmetro da rota
        const { id } = req.params;
        // Extrai os dados da requisição
        const data = await req.json() as any;
        // Adiciona um array com as chaves que foram atualizadas
        data.chavesAtualizadas = Object.keys(data);
        // Adiciona o ID aos dados de resposta
        data.id = id;
        // Adiciona a data/hora da atualização
        data.atualizadoEm = new Date().toLocaleDateString("pt-BR");
        // Retorna os dados com informações da atualização
        return Response.json(data);
      },

      // DELETE: Remove um recurso específico
      DELETE: (req, params) => {
        // Extrai o ID do parâmetro da rota
        const { id } = req.params;
        // Retorna mensagem de sucesso com status 200
        return new Response(`Recurso com id ${id} deletado`, { status: 200 });
      }
    }
    // ==================== FIM DO EXEMPLO ====================
  },

  // Manipulador padrão para rotas não definidas
  async fetch(req) {
    // Retorna erro 404 para qualquer rota não encontrada
    return new Response(`Not Found`, { status: 404 });
  },
});

// Exibe mensagem no console indicando que o servidor está rodando
console.log(`Server running at http://localhost:${server.port}`);

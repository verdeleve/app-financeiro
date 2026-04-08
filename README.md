<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="UTF-8">
<title>App Financeiro PRO - Corrigido</title>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>

<style>
* {
    box-sizing: border-box;
}

body {
    font-family: Arial, sans-serif;
    background: #0f172a;
    color: white;
    padding: 20px;
    max-width: 1200px;
    margin: 0 auto;
}

h1 { text-align: center; margin-bottom: 30px; }

.card {
    background: #1e293b;
    padding: 20px;
    border-radius: 16px;
    margin-bottom: 20px;
    box-shadow: 0 4px 6px rgba(0,0,0,0.3);
}

input, select, button {
    padding: 10px 12px;
    margin: 5px;
    border-radius: 10px;
    border: none;
    font-size: 14px;
}

button {
    background: #22c55e;
    color: white;
    cursor: pointer;
    transition: 0.2s;
    font-weight: bold;
}

button:hover {
    background: #16a34a;
    transform: scale(1.02);
}

.grid {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
}

.badge {
    padding: 8px 12px;
    border-radius: 10px;
    background: #334155;
    font-weight: bold;
}

.item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid #334155;
    padding: 10px 0;
    flex-wrap: wrap;
    gap: 10px;
}

.delete { background: #ef4444; }
.delete:hover { background: #dc2626; }
.edit { background: #f59e0b; }
.edit:hover { background: #d97706; }

.filtro-mes {
    background: #0f172a;
    padding: 10px;
    border-radius: 10px;
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
}

.filtro-mes label {
    font-weight: bold;
}

.filtro-mes input {
    background: #334155;
    color: white;
}

.btn-exportar {
    background: #3b82f6;
    margin-left: auto;
}

.btn-exportar:hover {
    background: #2563eb;
}

.total-grande {
    font-size: 2.5em;
    color: #22c55e;
    font-weight: bold;
}

@media (max-width: 600px) {
    body { padding: 10px; }
    .item { flex-direction: column; align-items: stretch; }
    .item div { text-align: right; }
}
</style>
</head>

<body>

<h1>💰 App Financeiro PRO</h1>

<div class="card">
    <h2>💰 Total do mês: <span class="total-grande" id="total">R$ 0,00</span></h2>
</div>

<div class="card filtro-mes">
    <label>📅 Mês/Ano:</label>
    <input type="month" id="mesSeletor">
    <button id="btnFiltrar" style="background:#3b82f6;">Filtrar</button>
    <button id="btnExportar" class="btn-exportar">📎 Exportar Dados</button>
    <button id="btnLimparTudo" style="background:#ef4444;">⚠️ Limpar Todos os Dados</button>
</div>

<div class="card">
    <h3>💳 Por forma de pagamento</h3>
    <div id="cards" class="grid"></div>
</div>

<div class="card">
    <h3>➕ Adicionar gasto</h3>
    <input id="desc" placeholder="Descrição (ex: Ifood, Uber, Posto...)">

    <select id="categoria">
        <option value="">Automático</option>
        <option>Alimentação</option>
        <option>Combustível</option>
        <option>Transporte</option>
        <option>Compras</option>
        <option>Assinatura</option>
        <option>Saúde</option>
        <option>Casa</option>
        <option>Outros</option>
    </select>

    <select id="cartao">
        <option>Nubank</option>
        <option>Ourocard</option>
        <option>Débito</option>
        <option>VR</option>
        <option>Dinheiro</option>
    </select>

    <input id="val" type="number" step="0.01" placeholder="Valor (R$)" min="0.01">

    <button id="btnAdd">➕ Adicionar</button>
    <button id="btnCancelar" style="background:#64748b;">✖️ Cancelar edição</button>
</div>

<div class="card">
    <h3>📂 Importar fatura (PDF)</h3>
    <input type="file" id="pdfInput" accept=".pdf">
    <small style="color:#94a3b8;">⚠️ O app tenta extrair valores, mas pode precisar de ajustes manuais.</small>
</div>

<div class="card">
    <h3>📊 Gráfico por categoria</h3>
    <canvas id="grafico" style="max-height: 300px;"></canvas>
</div>

<div class="card">
    <h3>📋 Gastos do período</h3>
    <div id="lista"></div>
    <div id="semDados" style="text-align:center; color:#94a3b8; padding:20px;">Nenhum gasto no período selecionado</div>
</div>

<script>

// ---------- DECLARAÇÃO DE VARIÁVEIS GLOBAIS ----------
let dados = JSON.parse(localStorage.getItem("dados")) || [];
let editId = null;              // ID do item sendo editado (em vez de índice)
let mesAtualFiltro = new Date().toISOString().slice(0, 7); // YYYY-MM

// Elementos DOM
const descEl = document.getElementById("desc");
const catEl = document.getElementById("categoria");
const cartaoEl = document.getElementById("cartao");
const valEl = document.getElementById("val");
const totalEl = document.getElementById("total");
const cardsDiv = document.getElementById("cards");
const pdfInput = document.getElementById("pdfInput");
const listaDiv = document.getElementById("lista");
const btnAdd = document.getElementById("btnAdd");
const btnCancelar = document.getElementById("btnCancelar");
const mesSeletor = document.getElementById("mesSeletor");
const btnFiltrar = document.getElementById("btnFiltrar");
const btnExportar = document.getElementById("btnExportar");
const btnLimparTudo = document.getElementById("btnLimparTudo");
const semDadosDiv = document.getElementById("semDados");
const graficoCanvas = document.getElementById("grafico");

let chartInstance = null; // para destruir gráfico antigo

// ---------- FUNÇÃO AUXILIAR: GERAR ID ÚNICO ----------
function gerarId() {
    return Date.now() + '-' + Math.random().toString(36).substr(2, 8);
}

// ---------- DETECÇÃO AUTOMÁTICA DE CATEGORIA ----------
function detectarCategoria(desc) {
    desc = desc.toLowerCase();

    if (desc.includes("mercado") || desc.includes("padaria") || desc.includes("restaurante") || desc.includes("ifood") || desc.includes("comida"))
        return "Alimentação";

    if (desc.includes("posto") || desc.includes("shell") || desc.includes("gasolina") || desc.includes("combustível"))
        return "Combustível";

    if (desc.includes("uber") || desc.includes("99") || desc.includes("taxi"))
        return "Transporte";

    if (desc.includes("netflix") || desc.includes("spotify") || desc.includes("disney") || desc.includes("amazon prime"))
        return "Assinatura";

    if (desc.includes("farmacia") || desc.includes("medico") || desc.includes("dentista"))
        return "Saúde";

    if (desc.includes("luz") || desc.includes("agua") || desc.includes("aluguel") || desc.includes("condominio"))
        return "Casa";

    if (desc.includes("roupa") || desc.includes("shopping") || desc.includes("amazon"))
        return "Compras";

    return "Outros";
}

// ---------- DETECTAR CARTÃO NO PDF ----------
function detectarCartaoPdf(linha) {
    let final = linha.match(/(\d{4})/);
    if (linha.includes("••••") && final)
        return "Nubank " + final[1];
    if (linha.toLowerCase().includes("ourocard"))
        return "Ourocard";
    return "Nubank";
}

// ---------- SALVAR NO LOCALSTORAGE E ATUALIZAR TELA ----------
function salvarEAtualizar() {
    localStorage.setItem("dados", JSON.stringify(dados));
    atualizarTela();
}

// ---------- ADICIONAR OU EDITAR GASTO ----------
function adicionarOuEditar() {
    let desc = descEl.value.trim();
    let categoria = catEl.value;
    let cartao = cartaoEl.value;
    let val = parseFloat(valEl.value);

    // Validações
    if (!desc) {
        alert("Digite uma descrição!");
        return;
    }
    if (isNaN(val) || val <= 0) {
        alert("Digite um valor válido (maior que zero)!");
        return;
    }

    // Se categoria for vazio ou "Automático", detectar automaticamente
    if (!categoria || categoria === "") {
        categoria = detectarCategoria(desc);
    }

    const hoje = new Date();
    const dia = hoje.getDate();
    const mes = hoje.getMonth() + 1;
    const ano = hoje.getFullYear();

    if (editId !== null) {
        // EDITAR: encontra o índice pelo ID
        const index = dados.findIndex(item => item.id === editId);
        if (index !== -1) {
            dados[index] = {
                ...dados[index],
                desc,
                categoria,
                cartao,
                val,
                dia,
                mes,
                ano
            };
        }
        editId = null;
        btnAdd.textContent = "➕ Adicionar";
        btnCancelar.style.display = "none";
    } else {
        // ADICIONAR
        dados.push({
            id: gerarId(),
            desc,
            categoria,
            cartao,
            val,
            dia,
            mes,
            ano
        });
    }

    // Limpar campos
    descEl.value = "";
    catEl.value = "";
    cartaoEl.value = "Nubank";
    valEl.value = "";

    salvarEAtualizar();
}

// ---------- EDITAR GASTO ----------
function editarGasto(id) {
    const item = dados.find(d => d.id === id);
    if (!item) return;

    descEl.value = item.desc;
    catEl.value = item.categoria;
    cartaoEl.value = item.cartao;
    valEl.value = item.val;

    editId = id;
    btnAdd.textContent = "💾 Salvar edição";
    btnCancelar.style.display = "inline-block";

    // Rolar para o formulário
    document.querySelector(".card:nth-of-type(3)").scrollIntoView({ behavior: "smooth" });
}

// ---------- REMOVER GASTO ----------
function removerGasto(id) {
    if (confirm("Tem certeza que deseja excluir este gasto?")) {
        dados = dados.filter(d => d.id !== id);
        if (editId === id) {
            // Se estava editando esse item, cancelar edição
            editId = null;
            btnAdd.textContent = "➕ Adicionar";
            btnCancelar.style.display = "none";
            descEl.value = "";
            catEl.value = "";
            valEl.value = "";
        }
        salvarEAtualizar();
    }
}

// ---------- CANCELAR EDIÇÃO ----------
function cancelarEdicao() {
    editId = null;
    btnAdd.textContent = "➕ Adicionar";
    btnCancelar.style.display = "none";
    descEl.value = "";
    catEl.value = "";
    valEl.value = "";
}

// ---------- PROCESSAR PDF ----------
function processarPdf(texto) {
    let linhas = texto.split(/(?=\d{2}\/\d{2})|(?=\d{1,2} [A-Z]{3})/);
    let novosGastos = 0;

    linhas.forEach(linha => {
        let data = linha.match(/(\d{2})\/(\d{2})/);
        let nubank = linha.match(/(\d{1,2}) [A-Z]{3}/);
        let valorMatch = linha.match(/R\$ ?([\d.,]+)/);
        
        if (!valorMatch) return;

        let valorStr = valorMatch[1].replace(/\./g, '').replace(',', '.');
        let valor = parseFloat(valorStr);
        if (isNaN(valor)) return;

        let desc = linha
            .replace(/R\$.*$/, "")
            .replace(/- Parcela.*$/, "")
            .replace(/.*\d{4}/, "")
            .replace(/[0-9\/]/g, '')
            .trim();

        if (desc.length < 2) desc = "Gasto identificado";

        let dia = 1;
        let mes = new Date().getMonth() + 1;
        let ano = new Date().getFullYear();

        if (data) {
            dia = parseInt(data[1]);
            mes = parseInt(data[2]);
        }
        if (nubank) {
            dia = parseInt(nubank[1]);
        }

        // Verifica se já existe gasto muito parecido (evitar duplicatas)
        const jaExiste = dados.some(d => 
            d.desc === desc && 
            Math.abs(d.val - valor) < 0.01 && 
            d.dia === dia && 
            d.mes === mes
        );

        if (!jaExiste) {
            dados.push({
                id: gerarId(),
                desc: desc,
                categoria: detectarCategoria(desc),
                cartao: detectarCartaoPdf(linha),
                val: valor,
                dia: dia,
                mes: mes,
                ano: ano
            });
            novosGastos++;
        }
    });

    if (novosGastos > 0) {
        alert(`✅ ${novosGastos} novos gastos importados do PDF!`);
        salvarEAtualizar();
    } else {
        alert("⚠️ Nenhum gasto novo encontrado no PDF (ou todos já existem).");
    }
}

// ---------- ATUALIZAR TELA COMPLETA ----------
function atualizarTela() {
    // Obter mês/ano selecionado
    const [anoSelecionado, mesSelecionado] = mesAtualFiltro.split('-').map(Number);
    
    // Filtrar dados pelo mês/ano
    const dadosFiltrados = dados.filter(d => 
        d.ano === anoSelecionado && d.mes === mesSelecionado
    );

    // Mostrar/esconder mensagem "sem dados"
    if (dadosFiltrados.length === 0) {
        semDadosDiv.style.display = "block";
        listaDiv.innerHTML = "";
    } else {
        semDadosDiv.style.display = "none";
        
        // Montar lista de gastos
        listaDiv.innerHTML = dadosFiltrados.map(item => `
            <div class="item">
                <span><strong>${item.dia}/${item.mes}</strong> - ${item.desc} <span style="color:#94a3b8;">(${item.cartao})</span></span>
                <span style="font-weight:bold;">R$ ${item.val.toFixed(2)}</span>
                <div>
                    <button class="edit" onclick="editarGasto('${item.id}')">✏️</button>
                    <button class="delete" onclick="removerGasto('${item.id}')">🗑️</button>
                </div>
            </div>
        `).join("");
    }

    // Total do mês
    const total = dadosFiltrados.reduce((soma, item) => soma + item.val, 0);
    totalEl.innerHTML = `R$ ${total.toFixed(2)}`;

    // Cards por forma de pagamento
    const cartoesMap = {};
    dadosFiltrados.forEach(item => {
        cartoesMap[item.cartao] = (cartoesMap[item.cartao] || 0) + item.val;
    });
    
    cardsDiv.innerHTML = Object.entries(cartoesMap).length > 0 
        ? Object.entries(cartoesMap).map(([k, v]) => 
            `<div class="badge">${k}: R$ ${v.toFixed(2)}</div>`
          ).join("")
        : '<div class="badge">Nenhum gasto no período</div>';

    // Gráfico por categoria
    const categoriaMap = {};
    dadosFiltrados.forEach(item => {
        categoriaMap[item.categoria] = (categoriaMap[item.categoria] || 0) + item.val;
    });

    if (chartInstance) {
        chartInstance.destroy();
    }

    if (Object.keys(categoriaMap).length > 0) {
        chartInstance = new Chart(graficoCanvas, {
            type: 'pie',
            data: {
                labels: Object.keys(categoriaMap),
                datasets: [{
                    data: Object.values(categoriaMap),
                    backgroundColor: ['#22c55e', '#3b82f6', '#f59e0b', '#ef4444', '#8b5cf6', '#ec489a', '#14b8a6', '#f97316'],
                    borderWidth: 0
                }]
            },
            options: {
                responsive: true,
                plugins: {
                    legend: { position: 'bottom', labels: { color: 'white' } }
                }
            }
        });
    } else {
        // Gráfico vazio
        chartInstance = new Chart(graficoCanvas, {
            type: 'pie',
            data: { labels: ['Sem dados'], datasets: [{ data: [1], backgroundColor: ['#334155'] }] },
            options: { plugins: { legend: { labels: { color: 'white' } } } }
        });
    }
}

// ---------- EXPORTAR DADOS ----------
function exportarDados() {
    const dataStr = JSON.stringify(dados, null, 2);
    const blob = new Blob([dataStr], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `financeiro_backup_${new Date().toISOString().slice(0,19)}.json`;
    a.click();
    URL.revokeObjectURL(url);
    alert("✅ Backup exportado com sucesso!");
}

// ---------- LIMPAR TODOS OS DADOS ----------
function limparTodosDados() {
    if (confirm("⚠️ ATENÇÃO: Isso vai apagar TODOS os seus gastos. Tem certeza?")) {
        dados = [];
        editId = null;
        salvarEAtualizar();
        cancelarEdicao();
        alert("Todos os dados foram removidos.");
    }
}

// ---------- IMPORTAR PDF ----------
async function importarPdf(file) {
    const reader = new FileReader();
    reader.onload = async function(event) {
        try {
            const pdf = await pdfjsLib.getDocument({ data: new Uint8Array(event.target.result) }).promise;
            let textoCompleto = "";
            for (let i = 1; i <= pdf.numPages; i++) {
                const page = await pdf.getPage(i);
                const content = await page.getTextContent();
                const pageText = content.items.map(item => item.str).join(" ");
                textoCompleto += pageText + "\n";
            }
            processarPdf(textoCompleto);
        } catch (error) {
            console.error("Erro ao ler PDF:", error);
            alert("Erro ao processar o PDF. Verifique se o arquivo é válido.");
        }
    };
    reader.readAsArrayBuffer(file);
}

// ---------- EVENTOS E INICIALIZAÇÃO ----------
function init() {
    // Configurar mês atual no seletor
    const hoje = new Date();
    const anoAtual = hoje.getFullYear();
    const mesAtual = String(hoje.getMonth() + 1).padStart(2, '0');
    mesAtualFiltro = `${anoAtual}-${mesAtual}`;
    mesSeletor.value = mesAtualFiltro;

    // Botões
    btnAdd.onclick = adicionarOuEditar;
    btnCancelar.onclick = cancelarEdicao;
    btnCancelar.style.display = "none";
    btnFiltrar.onclick = () => {
        mesAtualFiltro = mesSeletor.value;
        atualizarTela();
    };
    btnExportar.onclick = exportarDados;
    btnLimparTudo.onclick = limparTodosDados;
    
    pdfInput.onchange = (e) => {
        if (e.target.files.length > 0) {
            importarPdf(e.target.files[0]);
        }
        pdfInput.value = ""; // permitir importar o mesmo arquivo novamente
    };

    // Atualizar tela inicial
    atualizarTela();
}

// Iniciar quando a página carregar
init();

</script>
</body>
</html>

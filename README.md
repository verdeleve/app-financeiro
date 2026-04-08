<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Financeiro PRO | Dashboard</title>

    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>

    <style>
        :root {
            --bg: #0f172a;
            --card: #1e293b;
            --accent: #22c55e;
            --danger: #ef4444;
            --text: #f8fafc;
        }

        body {
            font-family: 'Segoe UI', Roboto, sans-serif;
            background: var(--bg);
            color: var(--text);
            margin: 0;
            padding: 20px;
        }

        .container { max-width: 900px; margin: 0 auto; }

        h1 { text-align: center; color: var(--accent); margin-bottom: 30px; }

        .card {
            background: var(--card);
            padding: 20px;
            border-radius: 16px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1);
        }

        .grid-header { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 15px; }

        select, input, button {
            padding: 12px;
            border-radius: 8px;
            border: 1px solid #334155;
            background: #0f172a;
            color: white;
            font-size: 14px;
        }

        button {
            background: var(--accent);
            border: none;
            cursor: pointer;
            font-weight: bold;
            transition: opacity 0.2s;
        }

        button:hover { opacity: 0.8; }
        button.btn-del { background: none; color: var(--danger); padding: 5px; }

        .stats-grid { display: flex; gap: 10px; flex-wrap: wrap; margin-top: 15px; }
        .badge { background: #334155; padding: 10px 15px; border-radius: 8px; font-size: 0.9em; }

        .form-input { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 10px; }

        .item {
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid #334155;
            padding: 12px 0;
        }

        canvas { max-height: 300px; }
        .charts-container { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; }
    </style>
</head>
<body>

<div class="container">
    <h1>💰 Financeiro PRO</h1>

    <div class="card">
        <div class="grid-header">
            <div>
                <span>Filtrar Mês:</span>
                <select id="filtroMes" onchange="atualizar()"></select>
            </div>
            <div style="text-align: right;">
                <small>Total no Mês</small>
                <h2 style="margin:0">R$ <span id="total">0,00</span></h2>
                <small id="comparacao" style="color: #94a3b8;"></small>
            </div>
        </div>
        <div id="cardsCartoes" class="stats-grid"></div>
    </div>

    <div class="card">
        <h3>➕ Novo Lançamento</h3>
        <div class="form-input">
            <input id="desc" placeholder="Descrição (Ex: Uber, Mercado...)">
            <select id="categoria">
                <option value="">Categoria Automática</option>
                <option>Alimentação</option>
                <option>Combustível</option>
                <option>Transporte</option>
                <option>Lazer</option>
                <option>Saúde</option>
                <option>Casa</option>
                <option>Outros</option>
            </select>
            <select id="cartao">
                <option>Nubank</option>
                <option>Inter</option>
                <option>Débito</option>
                <option>Dinheiro</option>
            </select>
            <input id="val" type="number" step="0.01" placeholder="Valor R$">
            <button onclick="add()">Adicionar</button>
        </div>
        <div style="margin-top: 15px;">
            <label style="font-size: 0.8em; color: #94a3b8;">Ou arraste um PDF de fatura:</label><br>
            <input type="file" id="pdfInput" accept="application/pdf">
        </div>
    </div>

    <div class="charts-container">
        <div class="card">
            <h3>📊 Por Categoria</h3>
            <canvas id="graficoCategoria"></canvas>
        </div>
        <div class="card">
            <h3>📈 Histórico Mensal</h3>
            <canvas id="graficoEvolucao"></canvas>
        </div>
    </div>

    <div class="card">
        <h3>📋 Extrato Detalhado</h3>
        <div id="lista"></div>
    </div>
</div>

<script>
// --- Configurações Iniciais ---
const pdfjsLib = window['pdfjs-dist/build/pdf'];
pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.worker.min.js';

let dados = JSON.parse(localStorage.getItem("dados_financeiros")) || [];
const mesesNomes = ["Jan", "Fev", "Mar", "Abr", "Mai", "Jun", "Jul", "Ago", "Set", "Out", "Nov", "Dez"];

// Preencher select de meses
const filtroMes = document.getElementById("filtroMes");
mesesNomes.forEach((mes, i) => {
    let opt = new Option(mes, i + 1);
    if (i === new Date().getMonth()) opt.selected = true;
    filtroMes.appendChild(opt);
});

// --- Lógica de Negócio ---

function detectarCategoria(desc) {
    desc = desc.toLowerCase();
    if (desc.includes("mercado") || desc.includes("ifood") || desc.includes("padaria")) return "Alimentação";
    if (desc.includes("posto") || desc.includes("shell") || desc.includes("ipiranga")) return "Combustível";
    if (desc.includes("uber") || desc.includes("99app") || desc.includes("onibus")) return "Transporte";
    if (desc.includes("farmacia") || desc.includes("hospital")) return "Saúde";
    if (desc.includes("netflix") || desc.includes("spotify") || desc.includes("steam")) return "Lazer";
    return "Outros";
}

function add() {
    const desc = document.getElementById("desc").value;
    const val = parseFloat(document.getElementById("val").value);
    const categoria = document.getElementById("categoria").value || detectarCategoria(desc);
    const cartao = document.getElementById("cartao").value;

    if (!desc || isNaN(val)) return alert("Preencha descrição e valor!");

    dados.push({
        id: Date.now(),
        desc, categoria, cartao, val,
        dia: new Date().getDate(),
        mes: parseInt(filtroMes.value)
    });

    document.getElementById("desc").value = "";
    document.getElementById("val").value = "";
    atualizar();
}

function remover(id) {
    dados = dados.filter(d => d.id !== id);
    atualizar();
}

// --- Processamento de PDF ---
document.getElementById("pdfInput").addEventListener('change', async (e) => {
    const file = e.target.files[0];
    const reader = new FileReader();
    reader.onload = async function() {
        const typedarray = new Uint8Array(this.result);
        const pdf = await pdfjsLib.getDocument(typedarray).promise;
        let fullText = "";
        
        for (let i = 1; i <= pdf.numPages; i++) {
            const page = await pdf.getPage(i);
            const content = await page.getTextContent();
            fullText += content.items.map(s => s.str).join(" ");
        }
        processarTextoPDF(fullText);
    };
    reader.readAsArrayBuffer(file);
});

function processarTextoPDF(texto) {
    // Regex genérica para capturar Data e Valor (R$)
    const regex = /(\d{2}\/\d{2})\s+(.*?)\s+R\$\s?([\d\.,]+)/g;
    let match;
    while ((match = regex.exec(texto)) !== null) {
        const [_, data, desc, valorStr] = match;
        const valor = parseFloat(valorStr.replace(".", "").replace(",", "."));
        const [dia, mes] = data.split("/").map(Number);

        dados.push({
            id: Math.random(),
            desc: desc.trim(),
            categoria: detectarCategoria(desc),
            cartao: "PDF Importado",
            val: valor,
            dia, mes
        });
    }
    atualizar();
}

// --- Renderização ---

function atualizar() {
    const mesAtual = parseInt(filtroMes.value);
    const dadosMes = dados.filter(d => d.mes === mesAtual);
    
    // Totais e Comparação
    const total = dadosMes.reduce((acc, cur) => acc + cur.val, 0);
    const totalMesAnterior = dados.filter(d => d.mes === (mesAtual === 1 ? 12 : mesAtual - 1))
                                 .reduce((acc, cur) => acc + cur.val, 0);
    
    document.getElementById("total").innerText = total.toLocaleString('pt-BR', {minimumFractionDigits: 2});
    
    if (totalMesAnterior > 0) {
        const diff = total - totalMesAnterior;
        const porcentagem = ((diff / totalMesAnterior) * 100).toFixed(0);
        document.getElementById("comparacao").innerText = `Mês ant: R$ ${totalMesAnterior.toFixed(2)} (${diff > 0 ? '↑' : '↓'} ${Math.abs(porcentagem)}%)`;
    }

    // Cartões
    const resumoCartao = {};
    dadosMes.forEach(d => resumoCartao[d.cartao] = (resumoCartao[d.cartao] || 0) + d.val);
    document.getElementById("cardsCartoes").innerHTML = Object.entries(resumoCartao)
        .map(([k, v]) => `<div class="badge"><strong>${k}:</strong> R$ ${v.toFixed(2)}</div>`).join("");

    // Lista
    document.getElementById("lista").innerHTML = dadosMes.sort((a,b) => b.dia - a.dia).map(d => `
        <div class="item">
            <div>
                <strong>${String(d.dia).padStart(2, '0')}/${String(d.mes).padStart(2, '0')}</strong> - ${d.desc}
                <br><small style="color:#94a3b8">${d.categoria} • ${d.cartao}</small>
            </div>
            <div>
                <span style="font-weight:bold">R$ ${d.val.toFixed(2)}</span>
                <button class="btn-del" onclick="remover(${d.id})">✕</button>
            </div>
        </div>
    `).join("");

    renderizarGraficos(dadosMes);
    localStorage.setItem("dados_financeiros", JSON.stringify(dados));
}

function renderizarGraficos(dadosMes) {
    const cores = ['#22c55e', '#3b82f6', '#f59e0b', '#ef4444', '#8b5cf6', '#ec4899', '#06b6d4'];
    
    // Gráfico Pizza
    const catData = {};
    dadosMes.forEach(d => catData[d.categoria] = (catData[d.categoria] || 0) + d.val);
    
    if (window.chart1) window.chart1.destroy();
    window.chart1 = new Chart(document.getElementById("graficoCategoria"), {
        type: 'doughnut',
        data: {
            labels: Object.keys(catData),
            datasets: [{ data: Object.values(catData), backgroundColor: cores, borderWidth: 0 }]
        },
        options: { plugins: { legend: { position: 'bottom', labels: { color: 'white' } } } }
    });

    // Gráfico Barras (Evolução)
    const evolucaoData = new Array(12).fill(0);
    dados.forEach(d => evolucaoData[d.mes - 1] += d.val);

    if (window.chart2) window.chart2.destroy();
    window.chart2 = new Chart(document.getElementById("graficoEvolucao"), {
        type: 'bar',
        data: {
            labels: mesesNomes,
            datasets: [{ label: 'Gastos R$', data: evolucaoData, backgroundColor: '#3b82f6' }]
        },
        options: { 
            scales: { 
                y: { ticks: { color: 'white' } },
                x: { ticks: { color: 'white' } }
            },
            plugins: { legend: { display: false } }
        }
    });
}

// Iniciar
atualizar();

</script>
</body>
</html>
        <option>VR</option>
    </select>

    <input id="val" type="number" placeholder="Valor">
    <button onclick="add()">Adicionar</button>
</div>

<div class="card">
    <h3>📂 Importar PDF</h3>
    <input type="file" id="pdfInput">
</div>

<div class="card">
    <canvas id="graficoCategoria"></canvas>
</div>

<div class="card">
    <canvas id="graficoEvolucao"></canvas>
</div>

<div class="card">
    <h3>📋 Gastos</h3>
    <div id="lista"></div>
</div>

<script>

let dados = JSON.parse(localStorage.getItem("dados")) || [];

// 🧠 Categoria automática
function detectarCategoria(desc) {
    desc = desc.toLowerCase();

    if (desc.includes("mercado") || desc.includes("padaria")) return "Alimentação";
    if (desc.includes("posto")) return "Combustível";
    if (desc.includes("uber")) return "Transporte";

    return "Outros";
}

// ➕ adicionar
function add() {
    let desc = descEl.value;
    let categoria = catEl.value || detectarCategoria(desc);
    let cartao = cartaoEl.value;
    let val = parseFloat(valEl.value);

    if (!desc || !val) return;

    let hoje = new Date();

    dados.push({
        desc, categoria, cartao, val,
        dia: hoje.getDate(),
        mes: hoje.getMonth()+1
    });

    limpar();
    atualizar();
}

// 🧹 limpar
function limpar() {
    descEl.value = "";
    catEl.value = "";
    valEl.value = "";
}

// 📂 importar PDF
pdfInput.addEventListener('change', async function(e) {
    const file = e.target.files[0];
    const reader = new FileReader();

    reader.onload = async function() {
        const pdf = await pdfjsLib.getDocument(new Uint8Array(this.result)).promise;

        for (let i=1;i<=pdf.numPages;i++) {
            let page = await pdf.getPage(i);
            let content = await page.getTextContent();
            let text = content.items.map(i=>i.str).join(" ");
            processar(text);
        }
    };

    reader.readAsArrayBuffer(file);
});

// 🧠 processar PDF
function processar(texto) {
    let linhas = texto.split(/(?=\d{2}\/\d{2})|(?=\d{1,2} [A-Z]{3})/);

    linhas.forEach(linha => {
        let data = linha.match(/(\d{2})\/(\d{2})/);
        let valorMatch = linha.match(/R\$ ?([\d,]+)/);
        if (!valorMatch) return;

        let valor = parseFloat(valorMatch[1].replace(",", "."));

        let desc = linha
            .replace(/R\$.*$/, "")
            .replace(/- Parcela.*$/, "")
            .replace(/.*\d{4}/, "")
            .trim();

        let dia = data ? parseInt(data[1]) : 1;
        let mes = data ? parseInt(data[2]) : new Date().getMonth()+1;

        dados.push({
            desc,
            categoria: detectarCategoria(desc),
            cartao: "Importado",
            val: valor,
            dia,
            mes
        });
    });

    atualizar();
}

// 🔄 atualizar
function atualizar() {

    let mesAtual = parseInt(filtroMes.value);
    let mesAnterior = mesAtual === 1 ? 12 : mesAtual - 1;

    let dadosMes = dados.filter(d => d.mes === mesAtual);
    let dadosAnt = dados.filter(d => d.mes === mesAnterior);

    let total = dadosMes.reduce((a,b)=>a+b.val,0);
    let totalAnt = dadosAnt.reduce((a,b)=>a+b.val,0);

    totalEl.innerText = total.toFixed(2);

    let diff = total - totalAnt;

    comparacao.innerText = totalAnt
        ? `Diferença: R$ ${diff.toFixed(2)} (${diff > 0 ? "↑" : "↓"})`
        : "";

    // 💳 cartões
    let cartoes = {};
    dadosMes.forEach(d=>{
        cartoes[d.cartao] = (cartoes[d.cartao]||0)+d.val;
    });

    cards.innerHTML = Object.entries(cartoes).map(([k,v]) =>
        `<div class="badge">${k}: R$ ${v.toFixed(2)}</div>`
    ).join("");

    // 📊 categoria
    let cat = {};
    dadosMes.forEach(d=>{
        cat[d.categoria]=(cat[d.categoria]||0)+d.val;
    });

    if(window.g1) window.g1.destroy();
    window.g1 = new Chart(graficoCategoria, {
        type: 'pie',
        data: {
            labels: Object.keys(cat),
            datasets: [{ data: Object.values(cat) }]
        }
    });

    // 📈 evolução
    let meses = {};
    dados.forEach(d=>{
        meses[d.mes] = (meses[d.mes]||0)+d.val;
    });

    if(window.g2) window.g2.destroy();
    window.g2 = new Chart(graficoEvolucao, {
        type: 'bar',
        data: {
            labels: Object.keys(meses),
            datasets: [{ data: Object.values(meses) }]
        }
    });

    // 📋 lista
    lista.innerHTML = dadosMes.map((d,i)=>`
        <div class="item">
            <span>${d.dia}/${d.mes} - ${d.desc} (${d.cartao}) - R$ ${d.val.toFixed(2)}</span>
        </div>
    `).join("");

    localStorage.setItem("dados", JSON.stringify(dados));
}

// iniciar no mês atual
filtroMes.value = new Date().getMonth()+1;

// DOM
const descEl = document.getElementById("desc");
const catEl = document.getElementById("categoria");
const cartaoEl = document.getElementById("cartao");
const valEl = document.getElementById("val");
const totalEl = document.getElementById("total");
const comparacao = document.getElementById("comparacao");
const cards = document.getElementById("cards");
const pdfInput = document.getElementById("pdfInput");
const lista = document.getElementById("lista");
const filtroMes = document.getElementById("filtroMes");

atualizar();

</script>

</body>
</html>

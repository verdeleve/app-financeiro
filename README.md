<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="UTF-8">
<title>App Financeiro PRO</title>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>

<style>
body {
    font-family: Arial;
    background: #0f172a;
    color: white;
    padding: 20px;
}

h1 { text-align: center; }

.card {
    background: #1e293b;
    padding: 20px;
    border-radius: 16px;
    margin-bottom: 20px;
}

select, input, button {
    padding: 10px;
    margin: 5px;
    border-radius: 10px;
    border: none;
}

button {
    background: #22c55e;
    color: white;
}

.grid {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
}

.badge {
    background: #334155;
    padding: 8px 12px;
    border-radius: 10px;
}

.item {
    display: flex;
    justify-content: space-between;
    border-bottom: 1px solid #334155;
    padding: 6px 0;
}
</style>
</head>

<body>

<h1>💰 App Financeiro</h1>

<div class="card">
    📅 Mês:
    <select id="filtroMes" onchange="atualizar()">
        <option value="1">Jan</option>
        <option value="2">Fev</option>
        <option value="3">Mar</option>
        <option value="4">Abr</option>
        <option value="5">Mai</option>
        <option value="6">Jun</option>
        <option value="7">Jul</option>
        <option value="8">Ago</option>
        <option value="9">Set</option>
        <option value="10">Out</option>
        <option value="11">Nov</option>
        <option value="12">Dez</option>
    </select>

    <h2>Total: R$ <span id="total">0</span></h2>
    <h3 id="comparacao"></h3>
</div>

<div class="card">
    <h3>💳 Por forma de pagamento</h3>
    <div id="cards" class="grid"></div>
</div>

<div class="card">
    <input id="desc" placeholder="Descrição">

    <select id="categoria">
        <option value="">Categoria</option>
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

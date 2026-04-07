<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="UTF-8">
<title>App Financeiro</title>

<!-- Chart -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<!-- PDF Reader -->
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

input, select, button {
    padding: 10px;
    margin: 5px;
    border-radius: 10px;
    border: none;
}

button {
    background: #22c55e;
    color: white;
    cursor: pointer;
}

.delete { background: red; }
.edit { background: orange; }

.item {
    display: flex;
    justify-content: space-between;
    border-bottom: 1px solid #334155;
    padding: 8px 0;
}
</style>
</head>

<body>

<h1>💰 App Financeiro</h1>

<div class="card">
    <h2>Total mês: R$ <span id="total">0</span></h2>
</div>

<div class="card">
    <input id="desc" placeholder="Descrição">

    <select id="categoria">
        <option value="">Selecionar categoria</option>
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
        <option>Pix</option>
        <option>Dinheiro</option>
    </select>

    <input id="val" type="number" placeholder="Valor">

    <button onclick="add()" id="btnAdd">Adicionar</button>
</div>

<!-- IMPORTAR PDF -->
<div class="card">
    <h3>📂 Importar fatura (PDF)</h3>
    <input type="file" id="pdfInput">
</div>

<div class="card">
    <canvas id="graficoCategoria"></canvas>
</div>

<div class="card">
    <canvas id="graficoMes"></canvas>
</div>

<div class="card">
    <h3>📋 Gastos do mês</h3>
    <div id="lista"></div>
</div>

<script>

let dados = JSON.parse(localStorage.getItem("dados")) || [];
let editIndex = null;

// 🧠 CATEGORIA AUTOMÁTICA (fallback)
function detectarCategoria(desc) {
    desc = desc.toLowerCase();

    if (desc.includes("mercado") || desc.includes("padaria") || desc.includes("restaurante") || desc.includes("ifood"))
        return "Alimentação";

    if (desc.includes("posto") || desc.includes("shell"))
        return "Combustível";

    if (desc.includes("uber"))
        return "Transporte";

    return "Outros";
}

// 💳 DETECTAR CARTÃO
function detectarCartao(linha) {
    if (linha.includes("••••")) return "Nubank";
    return "Ourocard";
}

// ➕ ADICIONAR / EDITAR
function add() {
    let desc = document.getElementById("desc").value;
    let categoria = document.getElementById("categoria").value;
    let cartao = document.getElementById("cartao").value;
    let val = parseFloat(document.getElementById("val").value);

    if (!desc || !val) return;

    let mes = new Date().getMonth() + 1;

    if (!categoria) categoria = detectarCategoria(desc);

    if (editIndex !== null) {
        dados[editIndex] = {desc, categoria, cartao, val, mes};
        editIndex = null;
        document.getElementById("btnAdd").innerText = "Adicionar";
    } else {
        dados.push({desc, categoria, cartao, val, mes});
    }

    limpar();
    atualizar();
}

// ✏️ EDITAR
function editar(i) {
    let d = dados[i];

    document.getElementById("desc").value = d.desc;
    document.getElementById("categoria").value = d.categoria;
    document.getElementById("cartao").value = d.cartao;
    document.getElementById("val").value = d.val;

    editIndex = i;
    document.getElementById("btnAdd").innerText = "Salvar";
}

// 🗑️ REMOVER
function remover(i) {
    dados.splice(i,1);
    atualizar();
}

// 🧹 LIMPAR
function limpar() {
    document.getElementById("desc").value = "";
    document.getElementById("categoria").value = "";
    document.getElementById("val").value = "";
}

// 📂 IMPORTAR PDF
document.getElementById('pdfInput').addEventListener('change', async function(e) {
    const file = e.target.files[0];
    const reader = new FileReader();

    reader.onload = async function() {
        const typedarray = new Uint8Array(this.result);
        const pdf = await pdfjsLib.getDocument(typedarray).promise;

        for (let i = 1; i <= pdf.numPages; i++) {
            const page = await pdf.getPage(i);
            const content = await page.getTextContent();

            const text = content.items.map(item => item.str).join(" ");
            processarTexto(text);
        }
    };

    reader.readAsArrayBuffer(file);
});

// 🧠 PROCESSAR TEXTO DO PDF
function processarTexto(texto) {

    let linhas = texto.split(/(?=\d{2}\/\d{2})|(?=\d{1,2} [A-Z]{3})/);

    linhas.forEach(linha => {

        let ouro = linha.match(/(\d{2})\/(\d{2}).*?R\$ ?([\d,]+)/);
        let nubank = linha.match(/(\d{1,2}) [A-Z]{3}.*?R\$ ?([\d,]+)/);

        let valor = 0;
        let desc = "";

        if (ouro) {
            valor = parseFloat(ouro[3].replace(",", "."));
            desc = linha.replace(/R\$.*$/, "").trim();
        }

        if (nubank) {
            valor = parseFloat(nubank[2].replace(",", "."));
            desc = linha
                .replace(/.*\d{4}/, "")
                .replace(/- Parcela.*$/, "")
                .replace(/R\$.*$/, "")
                .trim();
        }

        if (valor > 0) {
            dados.push({
                desc: desc,
                categoria: detectarCategoria(desc),
                cartao: detectarCartao(linha),
                val: valor,
                mes: new Date().getMonth() + 1
            });
        }

    });

    atualizar();
}

// 🔄 ATUALIZAR TELA
function atualizar() {

    let mesAtual = new Date().getMonth() + 1;
    let dadosMes = dados.filter(d => d.mes === mesAtual);

    let total = dadosMes.reduce((a,b)=>a+b.val,0);
    document.getElementById("total").innerText = total.toFixed(2);

    let lista = document.getElementById("lista");
    lista.innerHTML = dadosMes.map((d,i)=>`
        <div class="item">
            <span>${d.desc} (${d.cartao}) - R$ ${d.val.toFixed(2)} | ${d.categoria}</span>
            <div>
                <button class="edit" onclick="editar(${i})">✏️</button>
                <button class="delete" onclick="remover(${i})">X</button>
            </div>
        </div>
    `).join("");

    // gráfico categoria
    let cat = {};
    dadosMes.forEach(d=>{
        cat[d.categoria] = (cat[d.categoria]||0) + d.val;
    });

    if(window.g1) window.g1.destroy();
    window.g1 = new Chart(document.getElementById("graficoCategoria"), {
        type: 'pie',
        data: {
            labels: Object.keys(cat),
            datasets: [{ data: Object.values(cat) }]
        }
    });

    // gráfico mês
    let meses = {};
    dados.forEach(d=>{
        meses[d.mes] = (meses[d.mes]||0) + d.val;
    });

    if(window.g2) window.g2.destroy();
    window.g2 = new Chart(document.getElementById("graficoMes"), {
        type: 'bar',
        data: {
            labels: Object.keys(meses),
            datasets: [{ data: Object.values(meses) }]
        }
    });

    localStorage.setItem("dados", JSON.stringify(dados));
}

atualizar();

</script>

</body>
</html>

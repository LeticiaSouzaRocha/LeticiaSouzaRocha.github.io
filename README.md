<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Planejamento de Férias RH 2026-2027</title>
<script src="https://cdn.jsdelivr.net/npm/fullcalendar@6.1.15/index.global.min.js"></script>

<!-- ==================== FIREBASE ==================== -->
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-database-compat.js"></script>
<!-- ==================================================== -->

<style>
body{font-family:Segoe UI,Arial,sans-serif;background:#f4f6f9;margin:20px}
.top{background:#fff;padding:15px;border-radius:10px;margin-bottom:15px}
.grid{display:grid;grid-template-columns:repeat(3,1fr);gap:12px}
.month{background:#fff;border-radius:10px;padding:8px;box-shadow:0 1px 4px rgba(0,0,0,.1)}
.legend{display:flex;flex-wrap:wrap;gap:8px;margin-top:10px}
.person{cursor:pointer;padding:6px 10px;background:#eef;border-radius:6px}
.details{background:#fff;padding:10px;border-radius:8px;margin-top:10px}
button,input{padding:8px;margin:4px}
h2{margin-top:25px}
.fc .fc-toolbar{display:none}
#status{font-size:12px;color:#888;margin-left:8px}
#status.ok{color:#2e7d32}
#status.err{color:#c62828}
</style>
</head>
<body>
<div class='top'>
<h1>Planejamento de Férias <span id="status">conectando...</span></h1>
<input id='atuacao' placeholder='Atuação (Ex.: Sinistros)'>
<input id='nome' placeholder='Nome do colaborador'>
<button onclick='exportarTodos()'>👥 Exportar Todos</button>
<button onclick='exportarCalendario()'>📅 Exportar Calendário</button>
<button onclick='backup()'>💾 Backup</button>
<div id='legend' class='legend'></div>
<div id='details' class='details'>Clique em um colaborador para ver os períodos.</div>
</div>
<h2>2026</h2><div id='y2026' class='grid'></div>
<h2>2027</h2><div id='y2027' class='grid'></div>

<script>
/* ====================================================================
   CONFIGURAÇÃO DO FIREBASE
   Substitua os valores abaixo pelos dados do SEU projeto Firebase.
   Veja o passo a passo enviado junto com este arquivo para saber
   onde encontrar esses valores.
==================================================================== */
const firebaseConfig = {
  apiKey: "COLE_AQUI_SUA_API_KEY",
  authDomain: "SEU_PROJETO.firebaseapp.com",
  databaseURL: "https://SEU_PROJETO-default-rtdb.firebaseio.com",
  projectId: "SEU_PROJETO",
};

firebase.initializeApp(firebaseConfig);
const db = firebase.database();
const eventosRef = db.ref('feriasRH');
/* ==================================================================== */

let eventos = [];
let calendars = [];

function calcularDias(inicio, fim){
  const dataInicio = new Date(inicio);
  const dataFim = new Date(fim);
  return Math.floor((dataFim - dataInicio) / 86400000);
}

function cor(n){
  let h = 0;
  for (let c of n){ h = c.charCodeAt(0) + ((h<<5)-h); }
  return '#' + ((h>>>0) & 0xffffff).toString(16).padStart(6,'0');
}

/* ---------- Persistência: grava no Firebase (todos veem em tempo real) ---------- */
function save(){
  eventosRef.set(eventos).catch(err=>{
    console.error(err);
    alert('Erro ao salvar na nuvem. Verifique sua conexão. Detalhes no console.');
  });
}

/* ---------- Recebe atualizações em tempo real de qualquer usuário ---------- */
eventosRef.on('value', snapshot => {
  const val = snapshot.val();
  eventos = Array.isArray(val) ? val : (val ? Object.values(val) : []);
  atualizarTudo();
  setStatus(true);
}, err => {
  console.error(err);
  setStatus(false);
});

function setStatus(ok){
  const el = document.getElementById('status');
  el.textContent = ok ? '● conectado (tempo real)' : '● offline / erro de conexão';
  el.className = ok ? 'ok' : 'err';
}

/* ---------- Redesenha calendários e legenda a partir de `eventos` ---------- */
function atualizarTudo(){
  calendars.forEach(c=>{
    c.getEvents().forEach(ev=>ev.remove());
    eventos.forEach(e=>c.addEvent(e));
  });
  renderLegenda();
}

function renderLegenda(){
  let l = document.getElementById('legend');
  l.innerHTML = '';
  const grupos = {};
  eventos.forEach(e=>{
    const atuacao = e.atuacao || "Sem Atuação";
    if(!grupos[atuacao]) grupos[atuacao] = [];
    grupos[atuacao].push(e.title);
  });
  Object.keys(grupos).sort().forEach(atuacao=>{
    let bloco = document.createElement('div');
    bloco.style.marginRight = '20px';
    bloco.innerHTML = `<strong>${atuacao}</strong><br>`;
    [...new Set(grupos[atuacao])].sort().forEach(nome=>{
      let pessoa = document.createElement('div');
      pessoa.className = 'person';
      pessoa.innerHTML = `<span style="display:inline-block;width:12px;height:12px;background:${cor(nome)}"></span> ${nome}`;
      pessoa.onclick = ()=>mostrar(nome);
      bloco.appendChild(pessoa);
    });
    l.appendChild(bloco);
  });
}

function mostrar(nome){
  let periodos = eventos.filter(e=>e.title===nome);
  let atuacao = periodos[0]?.atuacao || "Não informada";
  let totalDias = 0;
  let html = `<h2>${nome}</h2><h4>${atuacao}</h4>`;

  periodos.forEach((e,i)=>{
    const diasPeriodo = calcularDias(e.start, e.end || e.start);
    totalDias += diasPeriodo;
    html += `
      <div style="padding:10px;border:1px solid #ddd;border-radius:8px;margin-bottom:10px;">
        <strong>Período ${i+1}</strong><br><br>
        ${e.start} até ${e.end||e.start}<br><br>
        📅 ${diasPeriodo} dias
      </div>`;
  });

  const saldo = 30 - totalDias;
  html += `
    <div style="background:#e8f5e9;padding:10px;border-radius:8px;margin-top:10px;">
      ✅ Total Programado: ${totalDias} dias
    </div>
    <br>
    <div style="background:#fff3cd;padding:10px;border-radius:8px;">
      ⚠ Saldo: ${saldo} dias
    </div>`;

  document.getElementById('details').innerHTML = html;

  document.querySelectorAll('.fc-event').forEach(ev=>{
    ev.style.opacity = ev.innerText.includes(nome) ? "1" : "0.20";
  });
}

function backup(){
  let b = new Blob([JSON.stringify(eventos)], {type:'application/json'});
  let a = document.createElement('a');
  a.href = URL.createObjectURL(b);
  a.download = 'backup-ferias.json';
  a.click();
}

function baixar(html, nome){
  let b = new Blob([html], {type:'text/html'});
  let a = document.createElement('a');
  a.href = URL.createObjectURL(b);
  a.download = nome;
  a.click();
}

function exportarTodos(){
  let g = {};
  eventos.forEach(e=>{
    if(!g[e.title]) g[e.title] = [];
    g[e.title].push(e);
  });

  let h = `<html><head><meta charset="UTF-8"><style>
    body{font-family:Segoe UI;margin:30px}
    .colab{margin-bottom:25px}
  </style></head><body><h1>Planejamento de Férias</h1>`;

  Object.keys(g).sort().forEach(nome=>{
    let total = 0;
    let atuacao = g[nome][0].atuacao || 'Não informada';
    h += `<div class="colab"><h2>${nome}</h2><h4>${atuacao}</h4>`;
    g[nome].forEach((e,i)=>{
      const diasPeriodo = calcularDias(e.start, e.end);
      total += diasPeriodo;
      h += `<p>Período ${i+1}<br>${e.start} até ${e.end}<br>📅 ${diasPeriodo} dias</p>`;
    });
    h += `<strong>✅ Total: ${total} dias</strong><br><strong>⚠ Saldo: ${30-total} dias</strong><hr></div>`;
  });

  h += `</body></html>`;
  baixar(h, 'Planejamento_Ferias.html');
}

function exportarCalendario(){
  let h = '<html><body><h1>Calendário Geral</h1>';
  eventos.forEach(e=>h += '<div style="background:'+e.color+';padding:8px;margin:4px"><b>'+e.title+'</b><br>'+e.start+' até '+(e.end||e.start)+'</div>');
  h += '</body></html>';
  baixar(h, 'Calendario_Geral.html');
}

function addMonth(container, id, date){
  let wrap = document.createElement('div');
  wrap.className = 'month';
  const [anoMes, mesMes, diaMes] = date.split('-').map(Number);
  const dataMes = new Date(anoMes, mesMes - 1, diaMes);
  const nomeMes = dataMes.toLocaleDateString('pt-BR', {month:'long', year:'numeric'});
  wrap.innerHTML = `<h3 style="text-align:center;margin:5px 0 10px 0;text-transform:capitalize;">${nomeMes}</h3><div id="${id}"></div>`;
  document.getElementById(container).appendChild(wrap);

  let cal = new FullCalendar.Calendar(document.getElementById(id), {
    initialView: 'dayGridMonth',
    initialDate: date,
    headerToolbar: {left:'title', center:'', right:''},
    height: 300,
    locale: 'pt-br',
    editable: true,
    selectable: true,
    events: [],
    select(info){
      let nome = document.getElementById('nome').value.trim();
      if(!nome){ alert('Digite o nome'); return; }
      if(eventos.filter(x=>x.title===nome).length >= 3){ alert('Máximo de 3 períodos'); return; }
      let ev = {
        id: Date.now()+Math.random()+'',
        title: nome,
        atuacao: document.getElementById('atuacao').value.trim(),
        start: info.startStr,
        end: info.endStr,
        color: cor(nome)
      };
      eventos.push(ev);
      save();
    },
    eventClick(info){
      if(confirm('Excluir período?')){
        eventos = eventos.filter(x=>x.id !== info.event.id);
        save();
      }
    },
    eventDrop(info){
      let e = eventos.find(x=>x.id===info.event.id);
      if(e){ e.start = info.event.startStr; e.end = info.event.endStr; save(); }
    },
    eventResize(info){
      let e = eventos.find(x=>x.id===info.event.id);
      if(e){ e.end = info.event.endStr; save(); }
    }
  });
  cal.render();
  calendars.push(cal);
}

['2026-09','2026-10','2026-11','2026-12'].forEach((m,i)=>addMonth('y2026','c26'+i,m+'-01'));
['2027-01','2027-02','2027-03','2027-04','2027-05','2027-06','2027-07','2027-08','2027-09','2027-10','2027-11','2027-12'].forEach((m,i)=>addMonth('y2027','c27'+i,m+'-01'));
</script>
</body>
</html>

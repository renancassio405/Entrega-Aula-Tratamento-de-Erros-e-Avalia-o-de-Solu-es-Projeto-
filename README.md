<!DOCTYPE html>
<html lang="pt-BR">

<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<title>Check-me</title>

<!-- FIREBASE -->
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>

<!-- EXCEL -->
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>

<style>

*{
  box-sizing:border-box;
}

body{
  margin:0;
  font-family:Arial;
  background:#0d0d0d;
  color:white;
}

.container{
  max-width:1400px;
  margin:20px auto;
  background:#1a1a1a;
  padding:20px;
  border-radius:12px;
  border:1px solid #333;
}

h1{
  text-align:center;
  color:#ff1e1e;
  margin-bottom:30px;
}

.form{
  display:grid;
  grid-template-columns:repeat(auto-fit,minmax(180px,1fr));
  gap:10px;
  margin-bottom:20px;
}

input{
  padding:12px;
  border-radius:8px;
  border:1px solid #333;
  background:#0d0d0d;
  color:white;
  outline:none;
}

input:focus{
  border-color:#ff1e1e;
}

button{
  padding:12px;
  border:none;
  border-radius:8px;
  cursor:pointer;
  font-weight:bold;
  transition:0.2s;
}

button:hover{
  transform:scale(1.03);
}

.red{
  background:#ff1e1e;
  color:white;
}

.gray{
  background:#333;
  color:white;
}

.green{
  background:#00aa55;
  color:white;
}

.blue{
  background:#0066ff;
  color:white;
}

.search-box{
  margin-top:20px;
}

table{
  width:100%;
  margin-top:20px;
  border-collapse:collapse;
  overflow:hidden;
  border-radius:10px;
}

th{
  background:#ff1e1e;
  padding:12px;
  font-size:14px;
}

td{
  padding:12px;
  border-bottom:1px solid #222;
  text-align:center;
}

tr:hover{
  background:#222;
}

.ok{
  text-decoration:line-through;
  opacity:0.5;
  background:#102010;
}

.badge{
  padding:6px 10px;
  border-radius:20px;
  font-size:12px;
  font-weight:bold;
}

.pending{
  background:#ff9900;
  color:black;
}

.done{
  background:#00cc66;
  color:white;
}

.actions{
  display:flex;
  gap:5px;
  justify-content:center;
}

@media(max-width:900px){

  table{
    display:block;
    overflow:auto;
  }

}

</style>
</head>

<body>

<div class="container">

<h1>🚗 CHECK-ME</h1>

<div class="form">

<input type="date" id="date" min="2024-01-01" max="2035-12-31">

<input type="time" id="time" min="08:00" max="17:00">

<input type="text" id="vehicle" placeholder="Veículo">

<input type="text" id="chassi" placeholder="Chassi">

<input type="text" id="seller" placeholder="Vendedor">

<input type="text" id="service" placeholder="Serviço">

<button class="red" onclick="add()">Salvar</button>

<button class="blue" onclick="autoFit()">
Reencaixe
</button>

<button class="green" onclick="exportExcel()">
Exportar Excel
</button>

</div>

<div class="search-box">

<input
type="text"
id="search"
placeholder="Buscar veículo, chassi ou vendedor..."
oninput="render()"
style="width:100%;"
>

</div>

<table>

<thead>
<tr>
<th>Status</th>
<th>Data</th>
<th>Hora</th>
<th>Veículo</th>
<th>Chassi</th>
<th>Vendedor</th>
<th>Serviço</th>
<th>Ações</th>
</tr>
</thead>

<tbody id="list"></tbody>

</table>

</div>

<script>

const firebaseConfig = {

  apiKey: "SUA_API_KEY",

  authDomain: "SEU_PROJETO.firebaseapp.com",

  projectId: "SEU_PROJETO",

  storageBucket: "SEU_PROJETO.appspot.com",

  messagingSenderId: "XXXX",

  appId: "XXXX"

};

firebase.initializeApp(firebaseConfig);

const db = firebase.firestore();

const dateInput = document.getElementById("date");
const timeInput = document.getElementById("time");
const vehicleInput = document.getElementById("vehicle");
const chassiInput = document.getElementById("chassi");
const sellerInput = document.getElementById("seller");
const serviceInput = document.getElementById("service");
const searchInput = document.getElementById("search");
const list = document.getElementById("list");

let cache = [];

async function add(){

  const date = dateInput.value.trim();
  const time = timeInput.value.trim();
  const vehicle = vehicleInput.value.trim();
  const chassi = chassiInput.value.trim();
  const seller = sellerInput.value.trim();
  const service = serviceInput.value.trim();

  if(!date || !time || !vehicle || !chassi || !seller || !service){

    alert("Preencha todos os campos");
    return;
  }

  if(time < "08:00" || time > "17:00"){

    alert("Horário permitido: 08:00 até 17:00");
    return;
  }

  // VERIFICAR CONFLITO
  const conflict = cache.find(i =>
    i.date === date &&
    i.time === time
  );

  if(conflict){

    alert("Já existe agendamento nesse horário");
    return;
  }

  try{

    await db.collection("concessionaria").add({

      date,
      time,
      vehicle,
      chassi,
      seller,
      service,
      ok:false,
      createdAt:new Date()

    });

    clearInputs();

    alert("Agendamento salvo");

  }catch(err){

    alert(err.message);

  }

}

db.collection("concessionaria")
.orderBy("date")
.orderBy("time")
.onSnapshot(snapshot=>{

  cache = [];

  snapshot.forEach(doc=>{

    cache.push({

      id:doc.id,
      ...doc.data()

    });

  });

  render();

});

function render(){

  let s = searchInput.value.toLowerCase();

  let html = "";

  cache
  .filter(i =>

    i.vehicle.toLowerCase().includes(s) ||
    i.chassi.toLowerCase().includes(s) ||
    i.seller.toLowerCase().includes(s) ||
    i.service.toLowerCase().includes(s)

  )

  .forEach(i=>{

    html += `

    <tr class="${i.ok ? 'ok' : ''}">

      <td>

        <span class="badge ${i.ok ? 'done':'pending'}">

          ${i.ok ? 'Concluído':'Pendente'}

        </span>

      </td>

      <td>${i.date}</td>

      <td>${i.time}</td>

      <td>${i.vehicle}</td>

      <td>${i.chassi}</td>

      <td>${i.seller}</td>

      <td>${i.service}</td>

      <td>

        <div class="actions">

          <button class="green"
          onclick="toggle('${i.id}', ${i.ok})">

            ✔

          </button>

          <button class="blue"
          onclick="moveSchedule('${i.id}')">

            ↔

          </button>

          <button class="red"
          onclick="del('${i.id}')">

            ✖

          </button>

        </div>

      </td>

    </tr>

    `;

  });

  list.innerHTML = html;

}

async function toggle(id,state){

  await db.collection("concessionaria")
  .doc(id)
  .update({

    ok:!state

  });

}

async function del(id){

  if(!confirm("Excluir agendamento?")) return;

  await db.collection("concessionaria")
  .doc(id)
  .delete();

}

function exportExcel(){

  const data = cache.map(i => ({

    Status:i.ok ? "Concluído":"Pendente",

    Data:i.date,

    Hora:i.time,

    Veículo:i.vehicle,

    Chassi:i.chassi,

    Vendedor:i.seller,

    Serviço:i.service

  }));

  const worksheet = XLSX.utils.json_to_sheet(data);

  const workbook = XLSX.utils.book_new();

  XLSX.utils.book_append_sheet(
    workbook,
    worksheet,
    "Agenda"
  );

  XLSX.writeFile(workbook,"check-me.xlsx");

}

function generateTimes(){

  const times = [];

  for(let h=8; h<=17; h++){

    times.push(`${String(h).padStart(2,"0")}:00`);

    if(h !== 17){

      times.push(`${String(h).padStart(2,"0")}:30`);

    }

  }

  return times;

}
  
async function autoFit(){

  const date = dateInput.value;

  if(!date){

    alert("Escolha uma data");
    return;
  }

  const horarios = generateTimes();

  const usados = cache
  .filter(i => i.date === date)
  .map(i => i.time);

  const livre = horarios.find(h => !usados.includes(h));

  if(!livre){

    alert("Agenda lotada");
    return;
  }

  timeInput.value = livre;

  alert("Horário disponível encontrado: " + livre);

}
async function moveSchedule(id){

  const novoHorario = prompt("Novo horário:");

  if(!novoHorario) return;

  const item = cache.find(i => i.id === id);

  const conflito = cache.find(i =>

    i.id !== id &&
    i.date === item.date &&
    i.time === novoHorario

  );

  if(conflito){

    alert("Horário já ocupado");
    return;
  }

  await db.collection("concessionaria")
  .doc(id)
  .update({

    time:novoHorario

  });

}

function clearInputs(){

  dateInput.value = "";
  timeInput.value = "";
  vehicleInput.value = "";
  chassiInput.value = "";
  sellerInput.value = "";
  serviceInput.value = "";

}

</script>

</body>
</html>

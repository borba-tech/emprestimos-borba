[controle.html](https://github.com/user-attachments/files/22393114/controle.html)
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Empréstimos Borba</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; background: #f5f5f5; }
    h1, h2 { text-align: center; }
    form, .loan, .details { background: #fff; padding: 15px; margin-bottom: 20px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,.1); }
    label { display: block; margin-top: 10px; }
    input, select, button { padding: 6px; margin-top: 4px; width: 100%; }
    table { width: 100%; border-collapse: collapse; margin-top: 10px; }
    th, td { border: 1px solid #ccc; padding: 6px; text-align: center; }
    th { background: #eee; }
    .paid { background: #d4edda; }
    .late { background: #f8d7da; }
    .actions { display: flex; gap: 5px; justify-content: center; flex-wrap: wrap; }
    #mainContent { display: none; }
  </style>
</head>
<body>

<div id="loginDiv">
  <h1>🔒 Empréstimos Borba</h1>
  <p>Digite a senha para acessar:</p>
  <input type="password" id="passwordInput" placeholder="Senha">
  <button onclick="checkPassword()">Entrar</button>
  <p id="errorMsg" style="color:red;"></p>
</div>

<div id="mainContent">
  <h1>📘 Empréstimos Borba</h1>

  <form id="loanForm">
    <h2>Novo Empréstimo</h2>
    <label>Nome do Cliente:<input required id="clientName"></label>
    <label>CPF:<input required id="clientCPF"></label>
    <label>Endereço:<input required id="clientAddress"></label>

    <label>Valor Emprestado (R$):<input type="number" required id="loanAmount"></label>
    <label>Taxa do Empréstimo (R$):<input type="number" required id="loanFee"></label>
    <label>Número de Períodos:<input type="number" required id="loanPeriods"></label>
    <label>Tipo de Período:
      <select id="loanType">
        <option value="daily">Diário</option>
        <option value="weekly">Semanal</option>
        <option value="monthly">Mensal</option>
      </select>
    </label>
    <label>Juros por Atraso (R$ fixo):<input type="number" required id="lateFee"></label>
    <label>Modo das Parcelas:
      <select id="parcelMode">
        <option value="auto">Automático (total ÷ períodos)</option>
        <option value="manual">Manual (digitar valores)</option>
      </select>
    </label>
    <button type="submit">Adicionar Empréstimo</button>
  </form>

  <div id="loansContainer"></div>
</div>

<script>
  const PASSWORD = 'cagt';
  const loginDiv = document.getElementById('loginDiv');
  const mainContent = document.getElementById('mainContent');
  const errorMsg = document.getElementById('errorMsg');

  function checkPassword(){
    const input = document.getElementById('passwordInput').value;
    if(input === PASSWORD){
      loginDiv.style.display = 'none';
      mainContent.style.display = 'block';
    }else{
      errorMsg.textContent = 'Senha incorreta!';
    }
  }

  const form = document.getElementById('loanForm');
  const container = document.getElementById('loansContainer');
  let loans = JSON.parse(localStorage.getItem('loans') || '[]');

  function saveLoans(){ localStorage.setItem('loans', JSON.stringify(loans)); }

  function renderLoans(){
    container.innerHTML = '';
    loans.forEach((loan, index) => {
      const div = document.createElement('div');
      div.className = 'loan';
      const totalPaid = loan.parcels.reduce((s,p)=>s+(p.paidAmount||0),0);
      const totalBase = loan.amount + loan.fee;
      const totalJuros = loan.parcels.reduce((s,p)=>s+(p.jurosPago||0),0);
      div.innerHTML = `
        <h3>${loan.clientName} (${loan.clientCPF})</h3>
        <p><b>Endereço:</b> ${loan.clientAddress}</p>
        <p><b>Valor Emprestado:</b> R$ ${loan.amount.toFixed(2)} | <b>Taxa:</b> R$ ${loan.fee.toFixed(2)}</p>
        <p><b>Total:</b> R$ ${totalBase.toFixed(2)} | <b>Pago:</b> R$ ${totalPaid.toFixed(2)} | <b>Juros:</b> R$ ${totalJuros.toFixed(2)} | <b>Saldo:</b> R$ ${(totalBase - totalPaid).toFixed(2)}</p>
        <button onclick="toggleDetails(${index})">Ver Detalhes</button>
        <div class="details" id="details-${index}" style="display:none;"></div>
      `;
      container.appendChild(div);
    });
  }

  function toggleDetails(i){
    const d = document.getElementById('details-'+i);
    d.style.display = d.style.display==='none'?'block':'none';
    if(d.innerHTML==='') renderDetails(i,d);
  }

  function renderDetails(i, container){
    const loan = loans[i];
    let html = `<table><tr><th>Parcela</th><th>Valor</th><th>Vencimento</th><th>Status</th><th>Ações</th></tr>`;
    loan.parcels.forEach((p, idx) => {
      const venc = new Date(p.dueDate);
      const hoje = new Date();
      let status = 'Pendente';
      let cls = '';
      if(p.paid){ status='Paga'; cls='paid'; }
      else if(hoje > venc){ status='Atrasada'; cls='late'; }
      html += `<tr class="${cls}"><td>${idx+1}</td>
        <td>R$ ${(p.value).toFixed(2)}</td>
        <td>${venc.toLocaleDateString()}</td>
        <td>${status}</td>
        <td class="actions">
          ${!p.paid ? `<button onclick="markPaid(${i},${idx})">Marcar Pago</button>`:''}
          <button onclick="printReceipt(${i},${idx})">Recibo</button>
        </td></tr>`;
    });
    html += `</table>
    <button onclick="printLoan(${i})">Recibo Completo</button>
    <button style="margin-top:10px;background:red;color:white;" onclick="deleteLoan(${i})">Excluir Empréstimo</button>`;
    container.innerHTML = html;
  }

  function markPaid(i, idx){
    const loan = loans[i];
    const p = loan.parcels[idx];
    const venc = new Date(p.dueDate);
    const hoje = new Date();
    let juros = 0;
    if(hoje > venc) juros = loan.lateFee;
    p.paid = true;
    p.jurosPago = juros;
    p.paidAmount = p.value + juros;
    saveLoans();
    renderLoans();
  }

  function printReceipt(i, idx){
    const loan = loans[i];
    const p = loan.parcels[idx];
    const w = window.open('', 'Recibo', 'width=600,height=400');
    w.document.write(`<h2>Recibo - Empréstimos Borba</h2>
      <p><b>Cliente:</b> ${loan.clientName}</p>
      <p><b>CPF:</b> ${loan.clientCPF}</p>
      <p><b>Endereço:</b> ${loan.clientAddress}</p>
      <p><b>Parcela:</b> ${idx+1}</p>
      <p><b>Valor Base:</b> R$ ${p.value.toFixed(2)}</p>
      <p><b>Juros Atraso:</b> R$ ${(p.jurosPago||0).toFixed(2)}</p>
      <p><b>Total Pago:</b> R$ ${(p.paidAmount||p.value).toFixed(2)}</p>
      <p><i>Obrigado pela preferência.</i></p>`);
    w.print();
  }

  function printLoan(i){
    const loan = loans[i];
    const w = window.open('', 'Recibo Completo', 'width=600,height=600');
    w.document.write(`<h2>Recibo Completo - Empréstimos Borba</h2>
      <p><b>Cliente:</b> ${loan.clientName}</p>
      <p><b>CPF:</b> ${loan.clientCPF}</p>
      <p><b>Endereço:</b> ${loan.clientAddress}</p>
      <hr>`);
    loan.parcels.forEach((p, idx)=>{
      w.document.write(`<p>Parcela ${idx+1}: R$ ${p.value.toFixed(2)} | Juros: R$ ${(p.jurosPago||0).toFixed(2)} | Pago: ${p.paid?'Sim':'Não'}</p>`);
    });
    w.document.write(`<hr><p><b>Total:</b> R$ ${(loan.amount+loan.fee).toFixed(2)}</p>`);
    w.print();
  }

  function deleteLoan(i){
    if(confirm('Tem certeza que deseja excluir este empréstimo?')){
      loans.splice(i,1);
      saveLoans();
      renderLoans();
    }
  }

  form.addEventListener('submit', e=>{
    e.preventDefault();
    const name = document.getElementById('clientName').value;
    const cpf = document.getElementById('clientCPF').value;
    const address = document.getElementById('clientAddress').value;
    const amount = parseFloat(document.getElementById('loanAmount').value);
    const fee = parseFloat(document.getElementById('loanFee').value);
    const periods = parseInt(document.getElementById('loanPeriods').value);
    const type = document.getElementById('loanType').value;
    const lateFee = parseFloat(document.getElementById('lateFee').value);
    const mode = document.getElementById('parcelMode').value;

    let total = amount + fee;
    let parcels = [];
    let today = new Date();
    for(let i=0;i<periods;i++){
      let venc = new Date(today);
      if(type==='daily') venc.setDate(venc.getDate()+(i+1));
      if(type==='weekly') venc.setDate(venc.getDate()+7*(i+1));
      if(type==='monthly') venc.setMonth(venc.getMonth()+(i+1));
      let value = mode==='auto'?(total/periods):0;
      parcels.push({value, dueDate:venc, paid:false});
    }

    let loan = {clientName:name, clientCPF:cpf, clientAddress:address, amount, fee, periods, type, lateFee, mode, parcels};
    loans.push(loan);
    saveLoans();
    renderLoans();
    form.reset();
  });

  renderLoans();
</script>
</body>
</html>

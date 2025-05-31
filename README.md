import { useState, useEffect } from "react";
import jsPDF from "jspdf";
import autoTable from "jspdf-autotable";
import { PieChart, Pie, Cell, Tooltip, ResponsiveContainer } from "recharts";

const COLORS = ["#34d399", "#f87171"];

export default function EmprestimosApp() {
  const [emprestimos, setEmprestimos] = useState([]);
  const [nome, setNome] = useState("");
  const [valor, setValor] = useState("");
  const [observacao, setObservacao] = useState("");
  const [vencimento, setVencimento] = useState("");
  const [filtro, setFiltro] = useState("todos");
  const [busca, setBusca] = useState("");
  const [editandoId, setEditandoId] = useState(null);

  useEffect(() => {
    const dados = localStorage.getItem("emprestimos");
    if (dados) setEmprestimos(JSON.parse(dados));
  }, []);

  useEffect(() => {
    localStorage.setItem("emprestimos", JSON.stringify(emprestimos));
  }, [emprestimos]);

  const adicionarEmprestimo = () => {
    if (!nome.trim() || !valor || parseFloat(valor) <= 0) return;
    if (editandoId) {
      setEmprestimos(
        emprestimos.map((e) =>
          e.id === editandoId
            ? { ...e, nome, valor: parseFloat(valor), observacao, vencimento }
            : e
        )
      );
      setEditandoId(null);
    } else {
      const novo = {
        id: Date.now(),
        nome,
        valor: parseFloat(valor),
        pago: false,
        data: new Date().toLocaleDateString(),
        observacao,
        vencimento,
      };
      setEmprestimos([novo, ...emprestimos]);
    }
    setNome("");
    setValor("");
    setObservacao("");
    setVencimento("");
  };

  const editarEmprestimo = (e) => {
    setNome(e.nome);
    setValor(e.valor.toString());
    setObservacao(e.observacao);
    setVencimento(e.vencimento || "");
    setEditandoId(e.id);
  };

  const marcarComoPago = (id) => {
    setEmprestimos(
      emprestimos.map((e) => (e.id === id ? { ...e, pago: true } : e))
    );
  };

  const totalEmprestado = emprestimos.reduce((s, e) => s + e.valor, 0);
  const totalPendente = emprestimos
    .filter((e) => !e.pago)
    .reduce((s, e) => s + e.valor, 0);
  const totalRecebido = emprestimos
    .filter((e) => e.pago)
    .reduce((s, e) => s + e.valor, 0);

  const gerarRelatorioCSV = () => {
    const linhas = [
      "Nome,Valor,Data,Observação,Vencimento,Situação",
      ...emprestimos.map(
        (e) =>
          `${e.nome},${e.valor.toFixed(2)},${e.data},${e.observacao || ""},${
            e.vencimento || ""
          },${e.pago ? "Pago" : "Pendente"}`
      ),
    ];
    const blob = new Blob([linhas.join("\n")], { type: "text/csv" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "relatorio_emprestimos.csv";
    a.click();
    URL.revokeObjectURL(url);
  };

  const gerarRelatorioPDF = () => {
    const doc = new jsPDF();
    doc.text("Relatório de Empréstimos", 14, 16);
    autoTable(doc, {
      startY: 20,
      head: [["Nome", "Valor", "Data", "Observação", "Vencimento", "Situação"]],
      body: emprestimos.map((e) => [
        e.nome,
        e.valor.toFixed(2),
        e.data,
        e.observacao || "",
        e.vencimento || "",
        e.pago ? "Pago" : "Pendente",
      ]),
    });
    doc.save("relatorio_emprestimos.pdf");
  };

  const dadosGrafico = [
    { name: "Recebido", value: totalRecebido },
    { name: "Pendente", value: totalPendente },
  ];

  const emprestimosFiltrados = emprestimos.filter((e) => {
    if (filtro === "pendente" && e.pago) return false;
    if (filtro === "pago" && !e.pago) return false;
    if (busca && !e.nome.toLowerCase().includes(busca.toLowerCase()))
      return false;
    return true;
  });

  return (
    <div className="p-4 max-w-xl mx-auto space-y-4">
      <h1 className="text-xl font-bold text-center text-blue-700">
        Controle de Empréstimos
      </h1>

      <div className="space-y-2">
        <input
          className="w-full border p-2 rounded"
          placeholder="Nome"
          value={nome}
          onChange={(e) => setNome(e.target.value)}
        />
        <input
          className="w-full border p-2 rounded"
          placeholder="Valor"
          type="number"
          value={valor}
          onChange={(e) => setValor(e.target.value)}
        />
        <input
          className="w-full border p-2 rounded"
          type="date"
          value={vencimento}
          onChange={(e) => setVencimento(e.target.value)}
        />
        <textarea
          className="w-full border p-2 rounded"
          placeholder="Observações"
          value={observacao}
          onChange={(e) => setObservacao(e.target.value)}
        />
        <button
          onClick={adicionarEmprestimo}
          className="w-full bg-blue-600 text-white p-2 rounded"
        >
          {editandoId ? "Atualizar" : "Adicionar"} Empréstimo
        </button>
      </div>

      <div className="text-sm space-y-1">
        <p>Total emprestado: R$ {totalEmprestado.toFixed(2)}</p>
        <p>Total pendente: R$ {totalPendente.toFixed(2)}</p>
        <p>Total recebido: R$ {totalRecebido.toFixed(2)}</p>
        <div className="flex gap-2 mt-2">
          <button onClick={gerarRelatorioCSV} className="bg-gray-200 p-2 rounded">
            Exportar CSV
          </button>
          <button onClick={gerarRelatorioPDF} className="bg-gray-200 p-2 rounded">
            Exportar PDF
          </button>
        </div>
      </div>

      <div className="mt-4">
        <ResponsiveContainer width="100%" height={200}>
          <PieChart>
            <Pie
              data={dadosGrafico}
              dataKey="value"
              cx="50%"
              cy="50%"
              outerRadius={60}
              label={({ name, percent }) =>
                `${name}: ${(percent * 100).toFixed(0)}%`
              }
            >
              {dadosGrafico.map((entry, index) => (
                <Cell
                  key={`cell-${index}`}
                  fill={COLORS[index % COLORS.length]}
                />
              ))}
            </Pie>
            <Tooltip />
          </PieChart>
        </ResponsiveContainer>

        <div className="flex gap-2 mt-4">
          <input
            className="w-full border p-2 rounded"
            placeholder="Buscar por nome"
            value={busca}
            onChange={(e) => setBusca(e.target.value)}
          />
          <select
            className="border p-2 rounded"
            value={filtro}
            onChange={(e) => setFiltro(e.target.value)}
          >
            <option value="todos">Todos</option>
            <option value="pendente">Pendentes</option>
            <option value="pago">Pagos</option>
          </select>
        </div>
      </div>

      <div className="space-y-2">
        {emprestimosFiltrados.map((e) => (
          <div
            key={e.id}
            className={`border p-3 rounded ${e.pago ? "opacity-60" : ""}`}
          >
            <p>
              <strong>{e.nome}</strong> — R$ {e.valor.toFixed(2)}
            </p>
            <p className="text-sm text-gray-500">Data: {e.data}</p>
            {e.vencimento && (
              <p className="text-sm text-red-600">Vencimento: {e.vencimento}</p>
            )}
            {e.observacao && <p className="text-sm">{e.observacao}</p>}
            <div className="flex gap-2 mt-2">
              {!e.pago && (
                <button
                  onClick={() => marcarComoPago(e.id)}
                  className="text-white bg-green-500 px-2 py-1 rounded text-sm"
                >
                  Marcar como Pago
                </button>
              )}
              <button
                onClick={() => editarEmprestimo(e)}
                className="text-sm border px-2 py-1 rounded"
              >
                Editar
              </button>
            </div>
            {e.pago && <p className="text-green-700 text-sm mt-1">Pago</p>}
          </div>
        ))}
      </div>
    </div>
  );
}

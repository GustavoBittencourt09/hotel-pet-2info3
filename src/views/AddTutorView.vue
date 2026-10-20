<script setup>
import { computed, ref } from 'vue';
import { useRouter } from 'vue-router';

const API_URL = 'http://localhost:3000';
const router = useRouter();

const tutor = ref({
  nome: '',
  cpfCnpj: '',
  telefone: '',
  dataNascimento: '',
  observacoes: '',
  endereco: {
    cep: '',
    rua: '',
    numero: '',
    bairro: '',
    cidade: '',
    estado: '',
  },
});

const tentouSalvar = ref(false);

function soNumeros(texto) {
  return texto.replace(/\D/g, '');
}

function mascaraTelefone(texto) {
  const n = soNumeros(texto).slice(0, 11);

  if (n.length <= 2) return n;
  if (n.length <= 6) return `(${n.slice(0, 2)}) ${n.slice(2)}`;
  if (n.length <= 10) return `(${n.slice(0, 2)}) ${n.slice(2, 6)}-${n.slice(6)}`;
  return `(${n.slice(0, 2)}) ${n.slice(2, 7)}-${n.slice(7)}`;
}

function mascaraCpfCnpj(texto) {
  const n = soNumeros(texto).slice(0, 14);

  if (n.length <= 11) {
    return n
      .replace(/(\d{3})(\d)/, '$1.$2')
      .replace(/(\d{3})(\d)/, '$1.$2')
      .replace(/(\d{3})(\d{1,2})$/, '$1-$2');
  }

  return n
    .replace(/(\d{2})(\d)/, '$1.$2')
    .replace(/(\d{3})(\d)/, '$1.$2')
    .replace(/(\d{3})(\d)/, '$1/$2')
    .replace(/(\d{4})(\d{1,2})$/, '$1-$2');
}

function mascaraCep(texto) {
  return soNumeros(texto)
    .slice(0, 8)
    .replace(/(\d{5})(\d)/, '$1-$2');
}

function digitouTelefone(e) {
  tutor.value.telefone = mascaraTelefone(e.target.value);
}

function digitouCpfCnpj(e) {
  tutor.value.cpfCnpj = mascaraCpfCnpj(e.target.value);
}

function digitouCep(e) {
  tutor.value.endereco.cep = mascaraCep(e.target.value);
  buscarCep();
}

async function buscarCep() {
  const cep = soNumeros(tutor.value.endereco.cep);

  if (cep.length !== 8) return;

  try {
    const resposta = await fetch(`https://viacep.com.br/ws/${cep}/json/`);
    const dados = await resposta.json();

    if (dados.erro) {
      alert('Não achei esse CEP, confere se digitou certo.');
      return;
    }

    tutor.value.endereco.rua = dados.logradouro || '';
    tutor.value.endereco.bairro = dados.bairro || '';
    tutor.value.endereco.cidade = dados.localidade || '';
    tutor.value.endereco.estado = dados.uf || '';
  } catch (erro) {
    console.error(erro);
    alert('Deu ruim na busca do CEP, tenta de novo.');
  }
}

const nomeOk = computed(() => tutor.value.nome.trim().length > 0);
const telefoneOk = computed(() => soNumeros(tutor.value.telefone).length >= 10);
const cpfOk = computed(() => {
  const tamanho = soNumeros(tutor.value.cpfCnpj).length;
  return tamanho === 0 || tamanho === 11 || tamanho === 14;
});
const formOk = computed(() => nomeOk.value && telefoneOk.value && cpfOk.value);

async function salvarTutor() {
  tentouSalvar.value = true;

  if (!formOk.value) return;

  const { endereco } = tutor.value;

  const dados = {
    nome: tutor.value.nome.trim(),
    telefone: soNumeros(tutor.value.telefone),
    endereco: {
      cep: soNumeros(endereco.cep),
      rua: endereco.rua.trim(),
      numero: endereco.numero.trim(),
      bairro: endereco.bairro.trim(),
      cidade: endereco.cidade.trim(),
      estado: endereco.estado.trim(),
    },
  };

  if (tutor.value.cpfCnpj) {
    dados.cpfCnpj = soNumeros(tutor.value.cpfCnpj);
  }

  if (tutor.value.dataNascimento) {
    dados.dataNascimento = tutor.value.dataNascimento;
  }

  if (tutor.value.observacoes.trim()) {
    dados.observacoes = tutor.value.observacoes.trim();
  }

  await fetch(`${API_URL}/tutores`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(dados),
  });

  router.push({ name: 'tutores' });
}
</script>

<template>
  <div>
    <header class="mb-4">
      <h1 class="display-5 fw-semibold">Cadastrar Tutores</h1>
      <p class="text-body-secondary">Cadastro de Tutores no sistema.</p>

      <RouterLink
        class="btn btn-outline-primary"
        :to="{ name: 'tutores' }"
      >
        Lista Tutores
      </RouterLink>
    </header>

    <form @submit.prevent="salvarTutor">
      <h2 class="h4 mb-2">Dados Gerais</h2>

      <div class="row g-3 mb-4">
        <div class="col-md-4">
          <label for="nome" class="form-label">Nome Completo:</label>
          <input
            id="nome"
            v-model="tutor.nome"
            type="text"
            class="form-control"
            :class="{ 'is-invalid': tentouSalvar && !nomeOk }"
          />
          <div class="invalid-feedback">Esse campo é obrigatório.</div>
        </div>

        <div class="col-md-4">
          <label for="cpfCnpj" class="form-label">CPF/CNPJ:</label>
          <input
            id="cpfCnpj"
            :value="tutor.cpfCnpj"
            type="text"
            class="form-control"
            :class="{ 'is-invalid': tentouSalvar && !cpfOk }"
            @input="digitouCpfCnpj"
          />
          <div class="invalid-feedback">CPF ou CNPJ incompleto.</div>
        </div>

        <div class="col-md-4">
          <label for="telefone" class="form-label">Telefone / WhatsApp:</label>
          <input
            id="telefone"
            :value="tutor.telefone"
            type="text"
            class="form-control"
            :class="{ 'is-invalid': tentouSalvar && !telefoneOk }"
            @input="digitouTelefone"
          />
          <div class="invalid-feedback">Informe um telefone com DDD.</div>
        </div>

        <div class="col-md-4">
          <label for="dataNascimento" class="form-label">Data de Nascimento</label>
          <input
            id="dataNascimento"
            v-model="tutor.dataNascimento"
            type="date"
            class="form-control"
          />
        </div>

        <div class="col-md-8">
          <label for="observacoes" class="form-label">Observações:</label>
          <textarea
            id="observacoes"
            v-model="tutor.observacoes"
            rows="2"
            class="form-control"
          ></textarea>
        </div>
      </div>

      <h2 class="h4 mb-2">Endereço</h2>

      <div class="row g-3 mb-4">
        <div class="col-md-4">
          <label for="cep" class="form-label">CEP</label>
          <input
            id="cep"
            :value="tutor.endereco.cep"
            type="text"
            class="form-control"
            @input="digitouCep"
          />
        </div>

        <div class="col-md-4">
          <label for="rua" class="form-label">Rua:</label>
          <input
            id="rua"
            v-model="tutor.endereco.rua"
            type="text"
            class="form-control"
          />
        </div>

        <div class="col-md-4">
          <label for="numero" class="form-label">Número:</label>
          <input
            id="numero"
            v-model="tutor.endereco.numero"
            type="text"
            class="form-control"
          />
        </div>

        <div class="col-md-4">
          <label for="bairro" class="form-label">Bairro</label>
          <input
            id="bairro"
            v-model="tutor.endereco.bairro"
            type="text"
            class="form-control"
          />
        </div>

        <div class="col-md-4">
          <label for="cidade" class="form-label">Cidade:</label>
          <input
            id="cidade"
            v-model="tutor.endereco.cidade"
            type="text"
            class="form-control"
          />
        </div>

        <div class="col-md-4">
          <label for="estado" class="form-label">Estado:</label>
          <input
            id="estado"
            v-model="tutor.endereco.estado"
            type="text"
            class="form-control"
          />
        </div>
      </div>

      <button type="submit" class="btn btn-success">Salvar Tutor</button>
    </form>
  </div>
</template>

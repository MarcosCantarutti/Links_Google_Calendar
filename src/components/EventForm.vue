<template>
    <div>
      <h3>Criar Eventos</h3>
      <button @click="adicionarEvento">+ Adicionar Evento</button>
  
      <div v-for="(evento, index) in eventos" :key="index" class="evento">
        <input v-model="evento.titulo" placeholder="Título" />
        <input v-model="evento.descricao" placeholder="Descrição do evento" />
        <input v-model="evento.inicio" type="datetime-local" />
        <input v-model="evento.fim" type="datetime-local" />
        <button @click="removerEvento(index)">Cancelar</button>
      </div>
  
      <button @click="criarEventos">
        {{ carregando ? "Criando..." : "Gerar Links" }}
      </button>
    </div>
  </template>
  
  <script setup>
  import { ref } from 'vue';
  
  const eventos = ref([]);
  const carregando = ref(false);
  
  const adicionarEvento = () => {
    eventos.value.push({ titulo: '', descricao: '', inicio: '', fim: '' });
  };
  
  const removerEvento = (index) => {
    eventos.value.splice(index, 1);
  };
  
  const criarEventos = () => {
    carregando.value = true;
    setTimeout(() => {
      carregando.value = false;
      // Emitir evento com os eventos gerados
      emit('eventosCriados', eventos.value);
      eventos.value = [];
    }, 1000);
  };
  
  defineProps({
    eventosCriados: Array,
  });
  </script>
  
  <style scoped>
  .evento {
    border: 1px solid #ccc;
    padding: 10px;
    margin-top: 10px;
  }
  </style>
  
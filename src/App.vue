<script setup>
import { ref, onMounted } from 'vue';
import supabase from './supabase'; 
import { format } from 'date-fns';
import { ptBR } from 'date-fns/locale';
import Swal from 'sweetalert2';


const user = ref(null);
const email = ref('');
const password = ref('');
const authError = ref('');
const events = ref([]);
const eventsCreated = ref([]);
const carregando = ref(false);
const currentPage = ref(1);
const eventsPerPage = 5;


// ?? Verificar usuário logado
onMounted(async () => {
  const { data: { user: loggedUser } } = await supabase.auth.getUser();
  user.value = loggedUser;
  if (user.value) {
    await loadingEvents(); // Carregar events do usuário ao entrar
  }
});

const formatDate = (dateString) => {
  const date = new Date(dateString);
  return format(date, 'eeee, dd MMMM yyyy HH:mm:ss', { locale: ptBR });
};

// ?? Login
const login = async () => {
  authError.value = '';
  const { data, error } = await supabase.auth.signInWithPassword({
    email: email.value,
    password: password.value
  });

  if (error) {
    authError.value = 'Falha ao logar. Verifique os dados.';
  } else {
    user.value = data.user;
    await loadingEvents(); // Carregar events após o login
  }
};

// ?? Logout
const logout = async () => {
  await supabase.auth.signOut();
  user.value = null;
  events.value = []; // Limpa os events ao deslogar
};

// ?? Adicionar novo event
const addEvent = () => {
  events.value.push({ title: '', description: '', start: '', end: '' });
};

// ?? Remover event da lista
const removerevent = (index) => {
  events.value.splice(index, 1);
};

const formatToGoogleCalendarDate = (date) => {
  const dt = new Date(date); // Criar objeto Date
  return dt.toISOString().replace(/[-:]/g, "").split(".")[0] + "Z"; 
};



// ?? Enviar events para o Supabase
const createEvents = async () => {
  if (events.value.length === 0) return;
  const userTimeZone = Intl.DateTimeFormat().resolvedOptions().timeZone;
  carregando.value = true;
  try {
    const eventsLinks = events.value.map(event => {
      const link = `https://www.google.com/calendar/render?action=TEMPLATE&text=${event.title}&details=${event.description}&dates=${formatToGoogleCalendarDate(event.start)}/${formatToGoogleCalendarDate(event.end)}&ctz=${userTimeZone}`;
      return {
        title: event.title,
        description: event.description,
        start: event.start,
        end: event.end,
        user_id: user.value.id, // Associa o event ao usuário logado
        link: link // Gera e associa o link para o event
      };
    });

    const { data, error } = await supabase
      .from('events_google_schedule')
      .insert(eventsLinks); // Inserir events com o link

    if (error) throw error;

    eventsCreated.value = data;
    events.value = []; // Limpa os events após criação
  } catch (error) {
    console.error('Erro ao criar events:', error.message);
  }
  carregando.value = false;

  loadingEvents();
};

const deleteEvent = async (eventId) => {
  if (!confirm('Tem certeza de que deseja deletar este evento?')) return;

  try {
    const { data, error } = await supabase
      .from('events_google_schedule')
      .delete()
      .eq('id', eventId); // Deleta o event com o id correspondente

    if (error) throw error;

    // Atualiza a lista de events após a exclusão
    eventsCreated.value = eventsCreated.value.filter(event => event.id !== eventId);
    alert('Event deletado com sucesso!');
  } catch (error) {
    console.error('Erro ao deletar event:', error.message);
  }

  loadingEvents();
};

// ?? Carregar events do usuário com paginação
const loadingEvents = async () => {
  if (!user.value) return;

  try {
    const { data, error } = await supabase
      .from('events_google_schedule')
      .select('*')
      .eq('user_id', user.value.id)
      .range((currentPage.value - 1) * eventsPerPage, currentPage.value * eventsPerPage - 1); // Paginação

    if (error) throw error;

    eventsCreated.value = data;
  } catch (error) {
    console.error('Erro ao carregar events:', error.message);
  }
};

// ?? Copiar link para área de transferência
const copiarLink = (link) => {
  navigator.clipboard.writeText(link).then(() => {
    Swal.fire({
      position: "top-end",
      title: 'Sucesso!',
      text: 'Link copiado para a área de transferência!',
      showConfirmButton: false, // Remove o botão
      timer: 1500, // Desaparece após 2 segundos
      timerProgressBar: true, // Exibe a barra de progresso
    });
  });
};

// ?? Alterar a página atual de eventos
const changePage = (page) => {
  currentPage.value = page;
  loadingEvents();
};
</script>

<template>
  <div class="max-w-6xl mx-auto p-6shadow-lg rounded-lg text-white  mt-20 border-2 border-white p-10 bg-stone-900">
    <h1 class="text-3xl  font-bold text-center mb-4">Gerador de Links de Eventos para o Google Calendar</h1>

    <p class="mb-6 text-gray-300 text-center">Forneça os dados do evento, e geraremos um link público que o usuário pode adicionar no Google Calendar.</p>

    <!-- ?? Login -->
    <div v-if="!user" class="space-y-4">
      <h2 class="text-xl font-semibold">Login</h2>
      <div class="space-y-2">
        <input v-model="email" type="email" placeholder="E-mail" class="w-full px-4 py-2 border rounded-lg" />
        <input v-model="password" type="password" placeholder="Password" class="w-full px-4 py-2 border rounded-lg" />
      </div>
      <button @click="login" class="w-full py-2 bg-blue-500 text-white rounded-lg">Login</button>
      <p v-if="authError" class="text-red-500 text-center">{{ authError }}</p>
    </div>

    <!-- ?? Dashboard -->
    <div v-else class="space-y-6">
      <div class="flex justify-start gap-2 items-center">
        <h2 class="text-xl font-semibold">Bem-vindo, {{ user.email }}</h2>
        <button @click="logout" class=" bg-red-600 text-white rounded-lg">Sair</button>
      </div>


      <h3 class="text-lg font-medium">Criando eventos - Ao utilizar no botão "Adicionar formulário de evento", é adicionado um formulário com dados do evento. Pode ser criado um ou mais de uma vez, em seguida será listado com os devidos links públicos para o usuário conseguir criar o evento no próprio calendário pessoal da conta.</h3>
      <button @click="addEvent" class="py-2 px-4 bg-green-600 text-white rounded-lg">Adicionar formulário de evento</button>

      <!-- Container flexível -->
      <div class="flex flex-wrap gap-2 justify-center items-center">
        <!-- Cada evento será uma "card" com um estilo responsivo -->
        <div v-for="(event, index) in events" :key="index" class="bg-gray-800 p-4 rounded-lg my-2 w-full sm:w-1/2 lg:w-1/3 xl:w-1/4">
          <h2 class="text-center mb-1">Evento</h2>

          <input v-model="event.title" placeholder="Título do evento" class="w-full p-2 border border-gray-300 rounded-lg mb-2" />
          <input v-model="event.description" placeholder="Descrição do evento" class="w-full p-2 border border-gray-300 rounded-lg mb-2" />
          
          <label for="start">Início:</label>
          <input v-model="event.start" id="start" type="datetime-local" class="w-full p-2 border border-gray-300 rounded-lg mb-2" />
          
          <label for="end">Fim:</label>
          <input v-model="event.end" type="datetime-local" id="end" class="w-full p-2 border border-gray-300 rounded-lg mb-4" />

          <button @click="removerevent(index)" class="py-2 px-4 bg-red-500 text-white rounded-lg">Remover formulário</button>
        </div>
      </div>

   

      <button :disabled="carregando" @click="createEvents" class="w-full py-2 bg-blue-500 text-white rounded-lg">
        {{ carregando ? "Criando..." : "Criar evento(s)" }}
      </button>

      <!-- ?? Lista de eventos criados -->
      <h3 class="text-lg font-medium mt-6">Listagem de eventos criados</h3>
      <ul class="space-y-4">
      <li v-for="event in eventsCreated" :key="event.id" 
          class="bg-gray-800 flex flex-col sm:flex-row gap-2 justify-between items-center p-4 rounded-lg">
        
        <!-- Conteúdo do evento -->
        <div class="flex flex-col flex-grow">
          <span class="text-lg font-semibold">Criado em {{ formatDate(event.created_at) }}</span>
          <span class="text-lg font-semibold">Título: {{ event.title }}</span>

          <span class="text-lg font-semibold">
            Descrição: 
            <span :class="{'line-clamp-2': !event.showMore}">{{ event.description }}</span>
            <button @click="event.showMore = !event.showMore" class="text-blue-300 mb-2">
              {{ event.showMore ? 'Ver menos' : 'Ver mais' }}
            </button>
          </span>

          <a :href="event.link" target="_blank" class="text-sky-500">Acessar o Evento</a>
        </div>

        <!-- Botões fixos -->
        <div class="flex flex-col sm:flex-row gap-2 flex-shrink-0">
          <button @click="copiarLink(event.link)" class="bg-sky-500 text-white rounded-lg w-32 py-2">
            Copiar link
          </button>
          <button @click="deleteEvent(event.id)" class="bg-red-500 text-white rounded-lg w-32 py-2">
            Deletar
          </button>
        </div>

      </li>
    </ul>


      <!-- ?? Paginação -->
      <div class="flex justify-center space-x-4 mt-6">
        <button v-if="currentPage > 1" @click="changePage(currentPage - 1)" class="py-2 px-4 bg-gray-300 rounded-lg">Anterior</button>
        <button @click="changePage(currentPage + 1)" class="py-2 px-4 bg-gray-300 rounded-lg">Próximo</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 500px;
  margin: auto;
  text-align: center;
}
input {
  display: block;
  width: 100%;
  margin-bottom: 10px;
  padding: 8px;
}
button {
  margin-top: 10px;
  padding: 10px;
  cursor: pointer;
}
.error {
  color: red;
}
.event {
  border: 1px solid #ccc;
  padding: 10px;
  margin-top: 10px;
}
</style>

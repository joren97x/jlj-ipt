<script setup>

import axios from 'axios'
import { ref, onMounted, watch } from 'vue'
import { useAuthStore } from '@/stores/auth-store'
import { useRouter } from 'vue-router'
import { io } from "socket.io-client"
import { formatDistance } from 'date-fns'

const router = useRouter()
const authStore = useAuthStore()
const messages = ref([])
const message = ref('')
const activeUsers = ref(0)
const tracks = ref([])
const q = ref('illit')
const currentSong = ref('3A2yGHWIzmGEIolwonU69h')
const play = ref(false)
const chosenMessage = ref(null)


const socket = io("http://localhost:3001");

socket.on("active users", (users) => {
	activeUsers.value = users
});

socket.on('chat message', () => {
	getAllMessages()
})

socket.on('delete message', () => {
	getAllMessages()
})

onMounted(async () => {
	getAllMessages()

	var client_id = '3986226f0e2d41b590779b28ce14a56a';
	var client_secret = '146395d6cb6d4e0093984a67141b5bed'

	axios.post('https://accounts.spotify.com/api/token', 
		`grant_type=client_credentials&client_id=${client_id}&client_secret=${client_secret}&scope=user-top-read`, 
		{
			headers: {
				'Content-Type': 'application/x-www-form-urlencoded'
			}
		}
	)
	.then(response => {
		console.log(response.data)
		authStore.access_token = response.data.access_token
		getTracks()
	})
	.catch(error => {
		console.log("in here 69")
		console.error(error);
	});

})

const getAllMessages = () => {
	axios.get('http://localhost:3000/', {
		headers: {
			Authorization: `Bearer ${authStore.token}`
		}
	})
	.then((res) => {
        console.log(res)
		messages.value = res.data
		// virtualScoller.value.scrollToIndex(messages.value.length - 1)
	}).catch((err) => {
		console.log("in here 5")
		console.error(err)
	})
}


const logout = () => {
	
	router.push('/login')
	authStore.logout()
}

const sendMessage = () => {
	axios.post('http://localhost:3000/send-message', {
		user_id: authStore.authUser.id,
		type: 0,
		message: message.value
	}, {
		headers: {
			Authorization: `Bearer ${authStore.token}`
		}
	}).then((res) => {
		console.log(res)
		if(res.status == 201) {
			socket.emit('chat message')
			message.value = ''
		}
	}).catch((err) => {
		console.log("in here 4")
		console.error(err)
	})
}

const playSong = (trackId) => {
	currentSong.value = trackId
	play.value = true
}

const shareSong = (trackId) => {
	axios.post('http://localhost:3000/send-message', {
		user_id: authStore.authUser.id,
		type: 1,
		message: trackId
	}, {
		headers: {
			Authorization: `Bearer ${authStore.token}`
		}
	}).then((res) => {
		console.log(res)
		if(res.status == 201) {
			socket.emit('chat message')
			message.value = ''
		}
	}).catch((err) => {
		console.log("in here 3")
		console.error(err)
	})
}

const getTracks = () => {
	axios.get(`https://api.spotify.com/v1/search?q=${q.value}&type=track&limit=5`, {
		headers: {
			Authorization: `Bearer ${authStore.access_token}`
		}
	})
	.then((res) => {
		tracks.value = res.data.tracks.items
		console.log(res)
		// q.value = ''
	})
	.catch((err) => {
		console.log("in here")
		console.log(err)
	})

}

</script>

<template>
    <div class="music-player">
      <div class="search-bar">
        <form @submit.prevent="getTracks">
            <input type="text" placeholder="Type a song name..." v-model="q" />
            <button @click="searchSongs">Search</button>
        </form>
      </div>
      <div class="songs-table">
        <table style="margin-bottom: 20px">
          <thead>
            <tr>
              <th>#</th>
              <th>Title</th>
              <th>Album</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(track, index) in tracks" :key="track.id">
              <td>{{ index + 1 }}</td>
              <td>
                <img :src="track.album.images[0].url" style="height: 30px; width: 30px">
                {{ track.name }}
                </td>
              <td>{{ track.album.name }}</td>
              <td>
                <button @click="playSong(track.id)">Play</button>
                <button @click="shareSong(track.id)">Share</button>
              </td>
            </tr>
          </tbody>
        </table>
        <iframe class="mt-4" id="spotifyIframe" style="border-radius:12px" :src="`https://open.spotify.com/embed/track/${currentSong}?utm_source=generator&autoplay=1`" width="100%" height="152" frameBorder="0" allowfullscreen="" allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture" loading="lazy"></iframe>
      </div>
      <div class="chat-section">
        <div class="chat-header">Chats</div>
        <div class="messages">
          <div class="message" v-for="message in messages" :key="message.id">
            <iframe class="" v-if="message.type" style="border-radius:12px" :src="`https://open.spotify.com/embed/track/${message.message}?utm_source=generator`" width="100%" height="152" frameBorder="0" allowfullscreen="" allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture" loading="lazy"></iframe>
            <div v-else>
                <div class="message-content">{{ message.message }}</div>
                <div class="message-time">{{ message.created_at }}</div>
            </div>
          </div>
        </div>
        <div class="message-input">
            <form @submit.prevent="sendMessage">
                <input type="text" v-model="message" placeholder="Type a message..." />
                <button type="submit">Send</button>
            </form> 
        </div>
      </div>
    </div>
</template>

<style scoped>
.music-player {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.search-bar {
  display: flex;
  margin-bottom: 20px;
}

.search-bar input {
  padding: 10px;
  font-size: 16px;
  margin-right: 10px;
}

.search-bar button {
  padding: 10px;
}

.songs-table {
  width: 80%;
  margin-bottom: 20px;
}

.songs-table table {
  width: 100%;
  border-collapse: collapse;
}

.songs-table th, .songs-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.songs-table th {
  background-color: #f2f2f2;
}

.chat-section {
  width: 80%;
}

.chat-header {
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 10px;
}

.messages {
  border: 1px solid #ddd;
  height: 200px;
  overflow-y: auto;
  padding: 10px;
  margin-bottom: 10px;
}

.message {
  margin-bottom: 10px;
}

.message-content {
  font-size: 16px;
}

.message-time {
  font-size: 12px;
  color: #999;
}

.message-input {
  display: flex;
}

.message-input input {
  flex: 1;
  padding: 10px;
  font-size: 16px;
}

.message-input button {
  padding: 10px;
}
</style>
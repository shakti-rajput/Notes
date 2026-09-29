```vue
UserCard.vue — Child

:"Vue, evaluate what is inside the quotes as JavaScript instead of treating it as a plain string."
setup - Vue automatically makes the variables/functions available to the template.

a)
<script setup lang="ts">
  // defineProps<{
  //   name:string
  //   age: number
  //   isOnline: boolean
  // }>()
  const props = withDefaults(defineProps<{name: string, age:number, isOnline?:boolean}>(), {isOnline:false})
</script>

<template>
  <div class ="user-card">
    <h3> {{name}}({{age}}) </h3>
    <p> {{isOnline ? 'Online' : 'Offline'}} </p>
  </div>
</template>


App.vue - Parent

<script lang="ts">
  import UserCard from './components/UserCard.vue'
</script>

<template>
  <UserCard name="Shakti" :age="30" :isOnline="true"/>
</template>


b)

Child:
<details>
  <summary>Solution - UserCard.vue (Updated Child)</summary>
  <script setup lang="ts">
    const props = withDefaults(defineProps<{name:string, age:number, isOnline:boolean}>())
    const emit = defineEmits<{(e:'toggle-status'):void}>()
  </script>

  <template>
    <div class="user-card">
      <h3>{{name}} ({{age}})</h3>
      <p>{{isOnline? 'Online':'Offline'}}</p>
      <button @click="emit('toggle-status')">Toggle Status</button>
    </div>
  </template>
</details>


Parent:
<details>
  <summary>Parent.vue(updated)</summary>

  <script setup lang="ts">
    import {ref} from 'vue'
    import UserCard from './UserCard.vue'

    const isOnline = ref(true)

    function handleToggle(){
      isOnline.value = !isOnline.value
    }
  </script>

  <template>
    <UserCard name="Shakti" :age="27" :isOnline="isOnline" @toggle-status="handleToggle" />
  </template>
  
</details>


c)
<details>
  <summary> </summary>
  <script>
    import {ref, computed} from 'vue'
    import UserCard from './UserCard.vue'

    const isOnline = ref(true)
    const name = ref('Shakti')

    const statusLabel = computed(() => { return `${name.value} is currently ${isOnline.value ? 'Online' : 'Offline'}` })

    function handleToggle() {
      isOnline.value = !isOnline.value
    }
  </script>
  
  <template>
    <p>{{statusLabel}}</p>
    <UserCard :name="name" :age'="27" :isOnline="isOnline" @toggle-status="handleToggle"/>
  </template>
</details>



d)

<details>
  <summary>
    
  </summary>
  <script setup lang="ts">
  import {ref} from 'vue'

  const count = ref(0)

  function increment(){
    count.value++
  }

  function decrement(){
    count.value--
  } 
  </script>
  
  <template>
    <div>
      <button @click='increment'>+</button>
      <span>{{count}}</span>
      <button @click='decrement'>-</button>
    </div>
  </template>
</details>


e)
<details>
  <summary>
    
  </summary>
  <script>
    
  </script>
  <template>
    <ul>
      <li v-for="fruit in fruits" :key="fruit">{{fruit}}</li>
    </ul>
  </template>
</details>

f)
<details>
  <summary>
    
  </summary>
  <template>
    <p v-if="isLoggedIn">Welcome back!</p>
    <p v-show="isLoading">Loading...</p>
  </template>
</details>

g)
<script setup lang="ts">
  import {ref} from 'vue'

  const items = ref(['Apple','Banana','Mango'])

  function addItem(){
    items.value.push('Orange')
  }
</script>

<template>
  <div>
    <h2>Shopping Cart</h2>
    <p>Total items:{{items.length}}</p>
    <ul>
      <li v-for="item in items" :key="item" >
        {{item}}
      </li>
    </ul>
    <button @click="addItem">Add Item</button>
    <p v-show="items.length===0"> Cart is empty </p>
  </div>
</template>

```

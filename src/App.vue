<script>
import * as todosApi from './api/todos';
import StatusFilter from './components/StatusFilter.vue';
import TodoItem from './components/TodoItem.vue';
import Message from './components/Message.vue';

export default {
  components: {
    StatusFilter,
    TodoItem,
    Message,
  },
  data() {
    return {
      todos: [],
      title: '',
      status: 'all',
      errorMessage: '',
      temporaryTodo: null,
      updatingTodo: null,
      deletingTodo: null,
      isClearCompletedClicked: false,
      isToggleAllClicked: false,
    };
  },
  computed: {
    isEveryTodoCompleted() {
      return this.todos.every(todo => todo.completed)
    },
    activeTodos() {
      return this.todos.filter(todo => !todo.completed);
    },
    completedTodos() {
      return this.todos.filter(todo => todo.completed);
    },
    visibleTodos() {
      switch (this.status) {
        case 'active':
          return this.activeTodos;

        case 'completed':
          return this.completedTodos;

        default:
          return this.todos;
      }
    }
  },
  mounted() {
    todosApi.getTodos()
      .then(({ data }) => this.todos = data)
      .catch(() => {
        this.$refs.errorMessage.show('Unable to load todos');
      });
  },
  methods: {
    addTodo() {
      if (this.title.trim()) {
        this.temporaryTodo = { id: 0, title: this.title, completed: false}

        todosApi.createTodo(this.title)
        .then(({ data }) => {
          this.todos = [...this.todos, data];
          this.title = '';
        })
        .catch(() => this.$refs.errorMessage.show('Unable to add todo'))
        .finally(() => this.temporaryTodo = null);
      } else {
        this.$refs.errorMessage.show('Title must be not empty');
      }  
    },
    updateTodo({ id, title, completed }) {
      this.updatingTodo = id;
      console.log(this.updatingTodo)
      todosApi.updateTodo({ id, title, completed })
        .then(({ data }) => {
          this.todos = this.todos.map(
            todo => todo.id !== id ? todo : data,
          );
        })
        .catch(() => this.$refs.errorMessage.show('Unable to update todo'))
        .finally(() => this.updatingTodo = null)
    },
    deleteTodo(todoId) {
      this.deletingTodo = todoId;

      todosApi.deleteTodo(todoId)
        .then(() => {
          this.todos = this.todos.filter(
            todo => todo.id !== todoId,
          );
        })
        .catch(() => this.$refs.errorMessage.show('Unable to delete todo'))
        .finally(() => this.deletingTodo = null)
    },
    toggleAll() {
      this.isToggleAllClicked = true;

      const toggleValue = this.activeTodos.length > 0 ? true : false;

      const updatePromises = this.todos.map(todo => {
        if (todo.completed !== toggleValue) {
          return todosApi.updateTodo({
            id: todo.id,
            title: todo.title,
            completed: toggleValue
          });
        } else {
          return Promise.resolve();
        }
      });

      Promise.all(updatePromises)
        .then(() => {
          this.todos = this.todos.map(todo => ({
            ...todo,
            completed: toggleValue
          }));
        })
        .catch(() => this.$refs.errorMessage.show('Unable to update todo'))
        .finally(() => this.isToggleAllClicked = false);
    },
    deleteCompletedTodos() {
      this.isClearCompletedClicked = true;

      const deletePromises = this.completedTodos.map(completedTodo => {
        return todosApi.deleteTodo(completedTodo.id)
      });

      Promise.all(deletePromises)
        .then(() => this.todos = this.todos.filter(todo => !todo.completed))
        .catch(() => this.$refs.errorMessage.show('Unable to delete todo'))
        .finally(() => this.isClearCompletedClicked = false)
    }
  },
};
</script>

<template>
  <div class="todoapp">
    <h1 class="todoapp__title">todos</h1>

    <div class="todoapp__content">
      <header class="todoapp__header">
        <button
          class="todoapp__toggle-all"
          :class="{ active: activeTodos.length === 0 }"
          @click="toggleAll()"
        ></button>

        <form @submit.prevent="addTodo">
          <input
            type="text"
            class="todoapp__new-todo"
            placeholder="What needs to be done?"
            v-model="title"
          />
        </form>
      </header>

      <TransitionGroup
        name="list"
        tag="section"
        class="todoapp__main"
      >
        <TodoItem
          v-for="todo, index of visibleTodos"
          :key="todo.id"
          :todo="todo"
          :updatingTodo="updatingTodo"
          :deletingTodo="deletingTodo"
          :isClearCompletedClicked="isClearCompletedClicked"
          :isToggleAllClicked="isToggleAllClicked"
          :isEveryTodoCompleted="isEveryTodoCompleted"
          @update="updateTodo"
          @delete="deleteTodo(todo.id)"
        />

        <TodoItem :todo="temporaryTodo" v-if="temporaryTodo"/>
      </TransitionGroup>

      <footer v-if="todos.length" class="todoapp__footer">
        <span class="todoapp__active-count">
          {{ activeTodos.length }} items left
        </span>

        <StatusFilter v-model="status" />

        <button
          v-if="completedTodos.length > 0"
          @click="deleteCompletedTodos"
          class="todoapp__clear-completed"
        >
          Clear completed
        </button>
      </footer>
    </div>

    <Message
      class="is-warning"
      ref="errorMessage"
    >
      <template #default="{ text }">
        <p>{{ text }}</p>
      </template>

      <template #header>
        <p>Server Error</p>
      </template>
    </Message>
  </div>
</template>

<style>
.list-enter-active,
.list-leave-active {
  max-height: 60px;
  transition: all 0.5s ease;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  max-height: 0;
  transform: scaleY(0);
}
</style>


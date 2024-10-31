<template>
  <div class="about">
    <h1>This is my page for noSQL</h1>
  </div>
</template>

<style>
@media (min-width: 1024px) {
  .about {
    min-height: 100vh;
    display: flex;
    align-items: center;
  }
}
</style>

<script lang="ts">
import PouchDB from 'pouchdb'; 

export default {
  data() {
    return {
      database: null as PouchDB.Database | null,
    };
  },

  methods: {
    initDatabase() {
        const db = new PouchDB('http://jm:pwd85@127.0.0.1:5986/blog_articles');
        if (db) {
            console.log("fconnected to collection 'blog'");
        } else {
            console.warn("Something went wrong");
        }
        //this.storage = db; 
    },
    
    fetchData() {
        //db.allDocs();
    }

  },

  mounted() {
    this.initDatabase()
  }

}
</script>

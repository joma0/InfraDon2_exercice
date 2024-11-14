<template>
  <div class="about">
    <h1>This is my page for noSQL</h1>
    <h2>Nombre de post: {{ postsData.length }}</h2>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <div class="ucfirst">{{ post.doc.post_name }}<em style="font-size: x-small;"
            v-if="post.doc.attributes?.creation_date">
            - {{ post.doc.attributes?.creation_date }}
          </em>
        </div>
      </li>
    </ul>
    <h2>Ajouter un document</h2>
    <button @click="createAndAddDemoDocument">Ajouter un document démo</button>
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
import {ref} from 'vue'; 
import PouchDB from 'pouchdb'; 

//structure d'un élément de la base de données
declare interface Post {
    _id: string,  
    doc: {
      post_name: string,
      post_content: string,
      attributes: {
        creation_date: string
      }
    }
}

export default {
  data() {
    return {
      //database: null as PouchDB.Database | null,
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
    };
  },

  methods: {
    initDatabase() {
        const db = new PouchDB('http://jm:pwd85@localhost:5984/post');
        if (db) {
            console.log("connected to collection 'post'");
            this.storage = db; 
        } else {
            console.warn("Something went wrong");
        }
    },
    
    //récupérer les données depuis PouchDB
    fetchData() {
        const storage = ref(this.storage);
        const self = this;
        if (storage.value) {
          (storage.value).allDocs({
            include_docs: true,
            attachments: true
          }).then(function (result: any) {
            console.log('fetchData success', result);
            self.postsData = result.rows;
          }.bind(this)).catch(function (error: any) {
            console.log('fetchData error', error);
        });
      } else {
        console.log('nothing in storage'); 
      }
    }, 

    createAndAddDemoDocument(){
      const document = {
        doc: {
          post_name: "article_demo",
          post_content: "Ceci est un article demo",
          attributes: {
            creation_date: "2024-09-09"
          }  
        }
      }
      this.addDocument(document);
    },

    addDocument (document: Omit<Post, '_id'>) {
      if(this.storage) {
        this.storage.post(document.doc).then((result) => {
          this.fetchData();
        }).catch((error) => {
          console.log('Failed to add document', error);
        });
        } else {
          console.warn('Database not initialized'); 
        }
    },

    updateDocument(id: string){
      if(this.storage){
        const storage = this.storage; 
        storage.get(id).then((document: any) => {
          document.post_content = "Ceci est le contenu modifié"; 
          return storage.put(document); 
        }).then(()=>{
          console.log('Document updated successfully');
        }).catch((err) => {
          console.log('Failed to update document', err);
        });        
      } else {
        console.warn('Database not initialized');
      }
    },

    deleteDocument(id: string){
      if(this.storage){
        const storage = this.storage; 
        storage.get(id).then((document: any) => {
          return storage.remove(document);
        }).then(() => {
          console.log('Document removed successfully'); 
        }).catch((err) => {
          console.log('Failed to remove document', err)
        })
      } else {
        console.warn('Database not initialized'); 
      }
    }

  },

  mounted() {
    this.initDatabase()
    this.fetchData()
    //this.updateDocument("709f1efa0635ecea06646df075016536"); 
    //this.deleteDocument("709f1efa0635ecea06646df07501e95d"); 
  }

}
</script>



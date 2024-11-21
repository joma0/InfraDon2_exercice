<template>
  <div class="about">
    <h1>This is my page for local DB</h1>
    <h2>Nombre de posts : {{ postsData.length }}</h2>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <div class="ucfirst">
          {{ post.post_name }}
          <em style="font-size: x-small" v-if="post.attributes?.creation_date">
            - {{ post.attributes?.creation_date }}
          </em>
        </div>
      </li>
    </ul>
    <button @click="addDocument(createDemoDocument())">Ajouter un document démo</button>
    <button @click="updateDistantDB()">Envoyer les données à la DB distante</button>
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
import { ref, watch } from 'vue'
import PouchDB from 'pouchdb'
import { isDeclarationStatement } from 'typescript'
import { jsx } from 'vue/jsx-runtime'

//Ceci est un test pour github

//définit la structure d'un document de la DB
declare interface Post {
  _id: string
  _rev: string
  post_name: string
  post_content: string
  attributes: {
    creation_date: string
  }
}

export default {
  data() {
    return {
      //url de la DB distante CouchDB
      remoteDB: 'http://jm:pwd85@localhost:5984/post',
      //tableau qui stocke les documents qui seront récupérés de la DB -> utilisé dans la partie template
      postsData: [] as Post[],
      //référence à l'instance de la DB locale
      db: null as PouchDB.Database | null
    }
  },

  methods: {
    //fonction pour initialiser la DB locale
    initDatabase() {
      //nom de la DB locale
      const localDB = 'post'
      //créer la DB locale (pas d'url, uniquement un nom)
      const $db = new PouchDB(localDB)
      if ($db) {
        console.log('Connected to collection: ' + $db.name)
      } else {
        console.warn('Something went wrong')
      }
      this.db = $db
    },

    //fonction pour récupérer les données de la DB locale (au début vide)
    fetchData() {
      //stocke la DB dans une variable
      const db = ref(this.db).value
      if (db) {
        db.allDocs
          .bind(this)({
            include_docs: true,
            attachments: true
          })
          .then((result: any) => {
            console.log('fetchData success =>', result.rows)
            this.postsData = result.rows.map((row: any) => row.doc)
          })
          .catch(function (error: any) {
            console.log('fetchData error', error)
          })
      }
    },

    //fonction pour créer un document démo
    //correspond à la fonction getFakePost() dans le code du prof
    createDemoDocument() {
      return {
        _id: new Date().toISOString(),
        post_name: 'demo_' + new Date().toISOString(),
        post_content: 'contenu du post',
        attributes: {
          creation_date: new Date().toISOString()
        }
      }
    },

    //fonction pour ajouter un document à la DB locale (en utilisant l'id)
    addDocument(document: Post) {
      const db = ref(this.db).value
      if (db) {
        db.put
          .bind(this)(document)
          .then(() => {
            console.log('Successfully added the document')
            this.fetchData()
          })
          .catch((error) => {
            console.log('Failed to add the document', error)
          })
      }
    },

    //fonction pour supprimer un document de la DB locale (en utilisant l'id et le rev)
    async deleteDocument(id: string, rev: string) {
      const db = ref(this.db).value
      if (db) {
        try {
          await db.remove(id, rev)
          console.log('Successfully deleted the document')
        } catch (error) {
          console.log('Failed to delete the document', error)
        }
      } else {
        console.log('Database not valid')
        return
      }
    },

    //fonction pour synchroniser les données depuis la DB distante vers la DB locale
    updateLocalDB() {
      const db = ref(this.db).value
      if (db) {
        db.replicate.from
          .bind(this)(this.remoteDB)
          .on('complete', () => {
            console.log('on replicate complete')
            this.fetchData()
          })
          .on('error', function (error) {
            console.log('error', error)
          })
      }
    },

    //fonction pour synchroniser les données depuis la DB locale vers la DB distante
    updateDistantDB() {
      const db = ref(this.db).value
      if (db) {
        db.replicate.to
          .bind(this)(this.remoteDB)
          .on('complete', () => {
            console.log('on replicate complete')
          })
          .on('error', function (error) {
            console.log('error', error)
          })
      }
    },

    //fonction qui permet d'écouter en continu la DB distante et appeler la fonction updateLocalDB à chaque chgmt
    watchRemoteDatabase() {
      const db = ref(this.db).value
      if (db) {
        //Ecouter en continu les changements dans la DB distante
        const remoteChanges = db.changes({
          live: true,
          since: 'now',
          include_docs: true
        })

        //
        remoteChanges.on('change', (change) => {
          console.log('Changmenet détecté; ', change)
          this.updateLocalDB()
        })
      } else {
        console.warn('Base de données locale non initialisée')
      }
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
    //this.updateLocalDB()
    this.watchRemoteDatabase()
  }
}
</script>

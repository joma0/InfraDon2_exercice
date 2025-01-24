<template>
  <div class="about">
    <h1>My local DB</h1>

    <div>
      <em style="font-size: x-small">Synchroniser avec la DB distante</em>
      <div>
        <button @click="updateDistantDB()">Envoyer</button>
        <button @click="updateLocalDB()">Réceptionner</button>
      </div>

      <em style="font-size: x-small">Générer des document demo</em>
      <div>
        <button @click="generateRandomDocuments(1)">1 document</button>
        <button @click="generateRandomDocuments(50)">50 documents</button>
      </div>

      <em style="font-size: x-small">Ajouter un document</em>
      <form id="addingDocument" @submit.prevent="createDocumentWithForm">
        <div>
          <label for="nom">Nom du post</label>
          <input id="name" type="text" v-model="newPost.name" />
        </div>
        <div>
          <label for="contenu">Contenu du post</label>
          <textarea id="content" type="text" v-model="newPost.content"></textarea>
        </div>
        <div>
          <label for="media">Médias</label>
          <input id="media" type="file" multiple @change="retrieveMediasOfForm" />
        </div>
        <button id="buttonAddDocument" type="submit">Ajouter un post</button>
      </form>
    </div>

    <h2>Mes posts</h2>
    <em style="font-size: x-small">{{ postsData.length }} posts</em>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <div class="ucfirst">
          {{ post.post_name }}
          <br />
          <em style="font-size: x-small" v-if="post.attributes?.creation_date">
            - {{ post.attributes?.creation_date }}
          </em>
          <br />
          {{ post.post_content }}
          <br />
          <!-- Affichage des médias -->
          {{ post._attachments }}
          <!-- <div v-if="post.doc._attachments"> -->
          <!-- <div v-for="attachment in post.doc._attachments" :key="attachmentId">
            <img :src="attachment.dataUrl" alt="Attachment" />
          </div> -->
          <!-- </div> -->
          <br />
          <button @click="editDocument(post._id)">Editer</button>
          <button @click="deleteDocument(post._id, post._rev)">Supprimer</button>
        </div>
      </li>
    </ul>
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
import pouchdbFind from 'pouchdb-find'
PouchDB.plugin(pouchdbFind)

//définit la structure d'un document de la DB
declare interface Post {
  _id: string
  _rev?: string
  _attachments?: {
    [attachmentId: string]: {
      content_type: string
      data: Blob | Buffer | string
    }
  }
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
      db: null as PouchDB.Database | null,
      //variable pour stocker les données du formulaire
      newPost: {
        name: '',
        content: '',
        media: [] as File[]
      },
      //tableau qui stocke les url des médias de chaque post
      mediaUrls: {} as { [postId: string]: string[] } // Clé = postId, Valeur = tableau des URLs des médias
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

      return Promise.resolve() // Si la base n'est pas initialisée, retourne une promesse vide
    },

    //fonction factory qui permet de générer des documents démo avec du contenu aléatoire
    //pour pouvoir tester la fonctionnalité des index, le titre et le contenu du post contiennent des mots choisis aléatoirement
    //dans une liste de 30 mots
    generateRandomDocuments(number: number) {
      const randomWords = [
        'sentier',
        'sommet',
        'escapade',
        'forêt',
        'panorama',
        'altitude',
        'boussole',
        'refuge',
        'trek',
        'nature',
        'soleil',
        'pluie',
        'vent',
        'brouillard',
        'orage',
        'neige',
        'température',
        'éclaircie',
        'grêle',
        'humidité',
        'snack',
        'énergie',
        'protéines',
        'fruits',
        'barres',
        'pique-nique',
        'hydratation',
        'chocolat',
        'sandwich',
        'thermos'
      ]

      for (let index = 1; index <= number; index++) {
        const randomPost: Post = {
          _id: new Date().toISOString() + index,
          post_name: 'Document demo : ' + randomWords[Math.floor(Math.random() * 30)],
          post_content:
            'Mon post parle des sujets suivants : ' +
            randomWords[Math.floor(Math.random() * 30)] +
            ' ' +
            randomWords[Math.floor(Math.random() * 30)] +
            ' ' +
            randomWords[Math.floor(Math.random() * 30)] +
            ' ' +
            randomWords[Math.floor(Math.random() * 30)],
          attributes: {
            creation_date: new Date().toISOString()
          }
        }
        this.addDocument(randomPost)
        //this.addMediasToPost(randomPost._id)
        console.log('post généré avec média')
      }
    },

    createDocumentWithForm() {
      const db = ref(this.db).value
      if (db && this.newPost.name && this.newPost.content) {
        const document: Post = {
          _id: new Date().toISOString(),
          post_name: this.newPost.name,
          post_content: this.newPost.content,
          attributes: {
            creation_date: new Date().toISOString()
          }
        }

        //Ajout du document sans média
        this.addDocument(document)

        //Ajout des médias s'il y en a
        if (this.newPost.media) {
          this.addMediasToPost(document._id, this.newPost.media)
        }
        console.log('post créé')
      }
    },

    //fonction pour ajouter un document à la DB locale (en utilisant l'id)
    addDocument(document: Post) {
      const db = ref(this.db).value
      if (db) {
        db.put
          .bind(this)(document)
          .then(() => db.get(document._id))
          .then(() => {
            console.log('Successfully added the document')
            this.fetchData()
          })
          .catch((error) => {
            console.log('Failed to add the document', error)
          })
      }
    },

    //fonction pour éditer un document
    editDocument(id: string) {
      if (this.db) {
        const db = this.db
        db.get(id)
          .then((document: any) => {
            document.post_content = document.post_content + 'Contenu modifié'
            return db.put(document)
          })
          .then(() => {
            console.log('Document updated successfully')
          })
          .catch((err) => {
            console.log('Failed to update document', err)
          })
      } else {
        console.warn('Database not initialized')
      }
    },

    //fonction pour supprimer un document de la DB locale (en utilisant l'id et le rev)
    async deleteDocument(id: string, rev?: string) {
      if (!rev) {
        console.warn('Cannot delete the document without _rev')
        return
      }

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

    //fonction pour récupérer les images du formulaires
    retrieveMediasOfForm(event: Event) {
      //input contient un objet FileList qui contient tous les fichiers sélectionnés
      const input = event.target as HTMLInputElement
      if (input.files && input.files[0]) {
        this.newPost.media = Array.from(input.files)
      }
    },

    //fonction pour lier un média à un post
    addMediasToPost(postId: string, files: File[], postRev?: string) {
      //const attachment = new Blob(['Mon media'], { type: 'text/plain' })
      const db = ref(this.db).value
      if (db) {
        db.get(postId)
          .then((doc: any) => {
            if (!doc._rev) {
              throw new Error('Document does not have valid _rev')
            }

            //Lier tous les médias au document
            const uploadPromises = files.map((file) =>
              db.putAttachment(postId, file.name, doc._rev, file, file.type)
            )

            //Attendre que tous les fichiers soient ajoutés
            return Promise.all(uploadPromises)
          })
          .then((result) => {
            console.log('Successfully added all medias', result)
          })
          .catch((err) => {
            console.error('Failed to add medias : ', err)
          })
      }
    },

    //fonction pour récupérer tous les médias liés à un post
    displayMedias(postId: string, postAttachments?: Object): Promise<string[]> {
      if (postAttachments === undefined) {
        console.warn('No attachments for this post')
        return Promise.resolve([])
      }

      const db = ref(this.db).value

      if (!db) {
        console.warn('Database not initialized')
        return Promise.resolve([])
      }

      const mediaPromises = Object.keys(postAttachments).map((key) =>
        db
          .getAttachment(postId, key)
          .then((blob) => {
            //convertir le Buffer en Blob si nécessaire
            if (blob instanceof Blob) {
              //convertir le blob en URL
              const mediaUrl = URL.createObjectURL(blob)
              return mediaUrl
            } else {
              throw new Error('Unsupported attachment type')
            }
          })
          .catch((err) => {
            console.log(`Error fetching attachment "${key}" for post "${postId}":`, err)
            return ''
          })
      )

      // Retourner toutes les URLs générées
      return Promise.all(mediaPromises)
    },

    //fonction pour supprimer un média par son nom
    deleteMedia(postId: string, mediaName: string) {
      const db = ref(this.db).value
      if (db) {
        db.get(postId)
          .then((doc: any) => {
            if (!doc._rev) {
              throw new Error('Document does not have valid _rev')
            }

            return db.removeAttachment(postId, mediaName, doc._rev)
          })
          .then(() => {
            console.log(`Media ${mediaName} deleted successfully`)
            this.fetchData()
          })
          .catch((err) => {
            console.error('Error deleting media', err)
          })
      }
    },

    //fonction pour créer un index
    createNewIndex() {
      const db = ref(this.db).value
      if (db) {
        db.createIndex({
          index: {
            fields: ['post_name', 'post_content']
          }
        })
          .then(function (result) {
            console.log('Successfully created the index')
          })
          .catch(function (err) {
            console.log(err)
          })
      } else {
        console.warn('Base de données non initialisée')
      }
    },

    //fonction pour faire une requête sur un index
    queryIndex(word: string) {
      const db = ref(this.db).value
      if (db) {
        db.find({
          selector: {
            $or: [{ post_name: { $regex: word } }, { post_content: { $regex: word } }]
          },
          fields: ['_id', 'post_name', 'post_content'],
          sort: ['post_name']
        })
          .then(function (result) {
            console.log('Sucessfull')
          })
          .catch(function (err) {
            console.log(err)
          })
      } else {
        console.warn('Base de données non initialisée')
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
    console.log(this.createNewIndex())
    this.queryIndex('chocolat')
    this.fetchData()
    this.watchRemoteDatabase()
  }
}
</script>

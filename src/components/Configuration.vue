<script setup>
import { ref, computed } from 'vue'
import draggable from 'vuedraggable'
import AddonItem from './AddonItem.vue'
import Authentication from './Authentication.vue'
import DynamicForm from './DynamicForm.vue'

const stremioAPIBase = "https://api.strem.io/api/"
let stremioAuthKey = ref('');
let addons = ref([])
let loadAddonsButtonText = ref('Load Addons')
const isLoadingAddons = ref(false)
const isSyncing = ref(false)
const isAuthenticated = ref(false)
const isLocked = computed(() => !isAuthenticated.value)

let isEditModalVisible = ref(false);
let currentManifest = ref({});
let currentEditIdx = ref(null);

// Feature toggles for manifest manipulation
const shouldRemoveSearchArtifacts = ref(false)
const shouldRemoveStandardCatalogs = ref(false)
const shouldRemoveMetaResource = ref(false)

// Restore functionality state
const backupFile = ref(null)
const backupFileName = ref('')
const isBackupValid = ref(false)
const isRestoring = ref(false)

// Addon management state
const isManageAddonVisible = ref(false)
const newAddonManifest = ref({
    name: '',
    description: '',
    logo: '',
    background: '',
    catalogs: []
})

const addonsCount = computed(() => Array.isArray(addons.value) ? addons.value.length : 0)

/**
 * Remove Cinemeta search catalogs and the 'search' extra from top catalogs.
 * - Deletes catalogs with id 'cinemeta.search' for types movie and series
 * - Removes extra entry with name 'search' from 'top' movie and series catalogs
 * Returns a summary of changes for logging.
 */
function removeCinemetaSearchArtifacts() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return {
            removedSearchMovieCatalog: false,
            removedSearchSeriesCatalog: false,
            removedTopMovieSearchExtra: false,
            removedTopSeriesSearchExtra: false,
        }
    }

    const cinemetaAddon = addons.value[cinemetaIndex]
    const manifest = cinemetaAddon.manifest || {}
    const originalCatalogs = Array.isArray(manifest.catalogs) ? manifest.catalogs : []

    const hadSearchMovie = originalCatalogs.some((c) => c && c.id === 'cinemeta.search' && c.type === 'movie')
    const hadSearchSeries = originalCatalogs.some((c) => c && c.id === 'cinemeta.search' && c.type === 'series')

    // Remove the search catalogs entirely
    let updatedCatalogs = originalCatalogs.filter((c) => !(c && c.id === 'cinemeta.search' && (c.type === 'movie' || c.type === 'series')))

    // Helper to remove 'search' from extra array
    function stripSearchExtra(catalog) {
        if (!catalog || !Array.isArray(catalog.extra)) return false
        const before = catalog.extra.length
        catalog.extra = catalog.extra.filter((e) => !(e && e.name === 'search'))
        return catalog.extra.length !== before
    }

    // Remove 'search' from top movie and top series extras
    const topMovie = updatedCatalogs.find((c) => c && c.id === 'top' && c.type === 'movie')
    const topSeries = updatedCatalogs.find((c) => c && c.id === 'top' && c.type === 'series')
    const removedTopMovieSearchExtra = stripSearchExtra(topMovie)
    const removedTopSeriesSearchExtra = stripSearchExtra(topSeries)

    // Persist changes back to the reactive state
    addons.value[cinemetaIndex].manifest = {
        ...manifest,
        catalogs: updatedCatalogs,
    }

    return {
        removedSearchMovieCatalog: hadSearchMovie,
        removedSearchSeriesCatalog: hadSearchSeries,
        removedTopMovieSearchExtra,
        removedTopSeriesSearchExtra,
    }
}

/**
 * Remove specific Cinemeta catalogs by id/type mapping:
 * - Popular => id 'top' for movie and series
 * - New => id 'year' for movie and series
 * - Featured => id 'imdbRating' for movie and series
 * Returns a summary of which targets were removed.
 */
function removeCinemetaStandardCatalogs() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return {
            removedPopularMovie: false,
            removedPopularSeries: false,
            removedNewMovie: false,
            removedNewSeries: false,
            removedFeaturedMovie: false,
            removedFeaturedSeries: false,
        }
    }

    const manifest = addons.value[cinemetaIndex].manifest || {}
    const originalCatalogs = Array.isArray(manifest.catalogs) ? manifest.catalogs : []

    const isPopularMovie = (c) => c && c.id === 'top' && c.type === 'movie'
    const isPopularSeries = (c) => c && c.id === 'top' && c.type === 'series'
    const isNewMovie = (c) => c && c.id === 'year' && c.type === 'movie'
    const isNewSeries = (c) => c && c.id === 'year' && c.type === 'series'
    const isFeaturedMovie = (c) => c && c.id === 'imdbRating' && c.type === 'movie'
    const isFeaturedSeries = (c) => c && c.id === 'imdbRating' && c.type === 'series'

    const hadPopularMovie = originalCatalogs.some(isPopularMovie)
    const hadPopularSeries = originalCatalogs.some(isPopularSeries)
    const hadNewMovie = originalCatalogs.some(isNewMovie)
    const hadNewSeries = originalCatalogs.some(isNewSeries)
    const hadFeaturedMovie = originalCatalogs.some(isFeaturedMovie)
    const hadFeaturedSeries = originalCatalogs.some(isFeaturedSeries)

    const updatedCatalogs = originalCatalogs.filter((c) => !(
        isPopularMovie(c) ||
        isPopularSeries(c) ||
        isNewMovie(c) ||
        isNewSeries(c) ||
        isFeaturedMovie(c) ||
        isFeaturedSeries(c)
    ))

    addons.value[cinemetaIndex].manifest = {
        ...manifest,
        catalogs: updatedCatalogs,
    }

    return {
        removedPopularMovie: hadPopularMovie,
        removedPopularSeries: hadPopularSeries,
        removedNewMovie: hadNewMovie,
        removedNewSeries: hadNewSeries,
        removedFeaturedMovie: hadFeaturedMovie,
        removedFeaturedSeries: hadFeaturedSeries,
    }
}

/**
 * removeMeta: Remove the 'meta' entry from Cinemeta's manifest.resources
 * Returns whether 'meta' was present and removed.
 */
function removeMeta() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return { removedMeta: false }
    }
    const manifest = addons.value[cinemetaIndex].manifest || {}
    const resources = Array.isArray(manifest.resources) ? manifest.resources : []
    const hadMeta = resources.includes('meta')
    if (!hadMeta) {
        return { removedMeta: false }
    }
    const updatedResources = resources.filter((r) => r !== 'meta')
    addons.value[cinemetaIndex].manifest = {
        ...manifest,
        resources: updatedResources,
    }
    return { removedMeta: true }
}

function loadUserAddons() {
    const key = stremioAuthKey.value
    if (!key) {
        console.error('No auth key provided')
        return
    }

    loadAddonsButtonText.value = 'Loading...'
    isLoadingAddons.value = true
    console.log('Loading addons...')

    const url = `${stremioAPIBase}addonCollectionGet`
    fetch(url, {
        method: 'POST',
        body: JSON.stringify({
            type: 'AddonCollectionGet',
            authKey: key,
            update: true,
        })
    }).then((resp) => {
        resp.json().then((data) => {
            console.log(data)
            if (!("result" in data) || data.result == null) {
                console.error("Failed to fetch user addons: ", data)
                alert('Failed to fetch user addons - are you sure you pasted the correct Stremio AuthKey?')
                return
            }
            addons.value = data.result.addons
        })
    }).catch((error) => {
        console.error('Error fetching user addons', error)
    }).finally(() => {
        loadAddonsButtonText.value = 'Load Addons'
        isLoadingAddons.value = false
    })
}

function syncUserAddons() {
    const key = stremioAuthKey.value
    if (!key) {
        console.error('No auth key provided')
        return
    }
    console.log('Syncing addons...')
    isSyncing.value = true

    // Apply selected manifest manipulations before syncing
    try {
        if (shouldRemoveSearchArtifacts.value) {
            const result = removeCinemetaSearchArtifacts()
            console.log('Applied removeCinemetaSearchArtifacts:', result)
        }
        if (shouldRemoveStandardCatalogs.value) {
            const result = removeCinemetaStandardCatalogs()
            console.log('Applied removeCinemetaStandardCatalogs:', result)
        }
        if (shouldRemoveMetaResource.value) {
            const result = removeMeta()
            console.log('Applied removeMeta:', result)
        }
    } catch (e) {
        console.error('Error applying selected manifest manipulations:', e)
    }

    const url = `${stremioAPIBase}addonCollectionSet`
    fetch(url, {
        method: 'POST',
        body: JSON.stringify({
            type: 'AddonCollectionSet',
            authKey: key,
            addons: addons.value,
        })
    }).then((resp) => {
        resp.json().then((data) => {
            if (!("result" in data) || data.result == null) {
                console.error("Sync failed: ", data)
                alert('Sync failed if unknown error')
                return
            } else if (!data.result.success) {
                alert("Failed to sync addons: " + data.result.error)
            } else {
                console.log("Sync complete: + ", data)
                alert('Sync complete!')
            }
        })
    }).catch((error) => {
        alert("Error syncing addons: " + error)
        console.error('Error fetching user addons', error)
    }).finally(() => {
        isSyncing.value = false
    })
}

function removeAddon(idx) {
    addons.value.splice(idx, 1)
}

function getNestedObjectProperty(obj, path, defaultValue = null) {
    try {
        return path.split('.').reduce((acc, part) => acc && acc[part], obj)
    } catch (e) {
        return defaultValue
    }
}

function setAuthKey(authKey) {
    stremioAuthKey.value = authKey
    console.log('AuthKey set to: ', stremioAuthKey.value)
}

function onAuthSuccess(authKey) {
    // Ensure our local state has the latest key, then auto-load addons
    setAuthKey(authKey)
    isAuthenticated.value = true
    console.log('Authentication successful, auto-loading addons...')
    loadUserAddons()
}

function openEditModal(idx) {
    isEditModalVisible.value = true;
    currentEditIdx.value = idx;
    currentManifest.value = { ...addons.value[idx].manifest };
    document.body.classList.add('modal-open');
}

function closeEditModal() {
    isEditModalVisible.value = false;
    currentManifest.value = {};
    currentEditIdx.value = null;
    document.body.classList.remove('modal-open');
}

function saveManifestEdit(updatedManifest) {
    try {
        addons.value[currentEditIdx.value].manifest = updatedManifest;
        closeEditModal();
    } catch (e) {
        alert('Failed to update manifest');
    }
}

function downloadBackup() {
    if (!addons.value || addons.value.length === 0) {
        alert('No configuration data available to backup');
        return;
    }

    try {
        // Only include the original addon data from the server
        const backupData = JSON.parse(JSON.stringify(addons.value)); // Deep clone to ensure original data

        const dataStr = JSON.stringify(backupData, null, 2);
        const dataBlob = new Blob([dataStr], { type: 'application/json' });
        
        const url = URL.createObjectURL(dataBlob);
        const link = document.createElement('a');
        link.href = url;
        link.download = `stremio-addons-backup-${new Date().toISOString().split('T')[0]}.json`;
        
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        
        URL.revokeObjectURL(url);
        
        console.log('Backup downloaded successfully');
    } catch (e) {
        console.error('Error creating backup:', e);
        alert('Failed to create backup file');
    }
}

function loadBackupFile() {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.json';
    
    input.onchange = (event) => {
        const file = event.target.files[0];
        if (!file) return;
        
        const reader = new FileReader();
        reader.onload = (e) => {
            try {
                let jsonData = JSON.parse(e.target.result);
                
                // Check if it's the alternative format (object with "addons" property)
                if (!Array.isArray(jsonData) && jsonData && typeof jsonData === 'object' && Array.isArray(jsonData.addons)) {
                    console.log('Detected Sidekick backup format');
                    jsonData = jsonData.addons; // Extract the addons array
                }
                
                // Validate that it's an array and has addon-like structure
                if (!Array.isArray(jsonData)) {
                    throw new Error('Backup file must contain an array of addons');
                }
                
                // Basic validation - check if items have manifest property
                const hasValidStructure = jsonData.every(item => 
                    item && typeof item === 'object' && item.manifest
                );
                
                if (!hasValidStructure) {
                    throw new Error('Invalid addon structure in backup file');
                }
                
                // Store the validated data
                backupFile.value = jsonData;
                backupFileName.value = file.name;
                isBackupValid.value = true;
                
                console.log(`Loaded backup file: ${file.name} with ${jsonData.length} addons`);
            } catch (error) {
                console.error('Error parsing backup file:', error);
                alert(`Invalid backup file: ${error.message}`);
                
                // Reset state on error
                backupFile.value = null;
                backupFileName.value = '';
                isBackupValid.value = false;
            }
        };
        
        reader.readAsText(file);
    };
    
    input.click();
}

function restoreFromBackup() {
    if (!backupFile.value || !isBackupValid.value) {
        alert('No valid backup file loaded');
        return;
    }
    
    const key = stremioAuthKey.value;
    if (!key) {
        console.error('No auth key provided');
        return;
    }
    
    console.log('Restoring addons from backup...');
    isRestoring.value = true;
    
    const url = `${stremioAPIBase}addonCollectionSet`;
    fetch(url, {
        method: 'POST',
        body: JSON.stringify({
            type: 'AddonCollectionSet',
            authKey: key,
            addons: backupFile.value, // Use backup data instead of current addons
        })
    }).then((resp) => {
        resp.json().then((data) => {
            if (!("result" in data) || data.result == null) {
                console.error("Restore failed: ", data);
                alert('Restore failed with unknown error');
                return;
            } else if (!data.result.success) {
                alert("Failed to restore addons: " + data.result.error);
            } else {
                console.log("Restore complete: ", data);
                alert(`Successfully restored ${backupFile.value.length} addons from ${backupFileName.value}!`);
                
                // Update local state with restored data
                addons.value = JSON.parse(JSON.stringify(backupFile.value));
                
                // Reset backup state after successful restore
                backupFile.value = null;
                backupFileName.value = '';
                isBackupValid.value = false;
            }
        });
    }).catch((error) => {
        alert("Error restoring addons: " + error);
        console.error('Error restoring addons', error);
    }).finally(() => {
        isRestoring.value = false;
    });
}

function toggleAddonManagement() {
    isManageAddonVisible.value = !isManageAddonVisible.value;
    if (!isManageAddonVisible.value) {
        resetNewAddonForm();
    }
}

function resetNewAddonForm() {
    newAddonManifest.value = {
        name: '',
        description: '',
        logo: '',
        background: '',
        catalogs: []
    };
}

function saveNewAddon(manifest) {
    try {
        // Create a new addon object in the format expected by Stremio
        const newAddon = {
            manifest: { ...manifest },
            transportUrl: '', // Will be set by user or generated
            flags: {}
        };
        
        // Add to the addons list
        if (!Array.isArray(addons.value)) {
            addons.value = [];
        }
        addons.value.push(newAddon);
        
        console.log('New addon created:', newAddon);
        alert(`Addon "${manifest.name}" created successfully! Remember to sync to apply changes.`);
        
        // Reset form and hide management UI
        resetNewAddonForm();
        isManageAddonVisible.value = false;
    } catch (e) {
        console.error('Error creating new addon:', e);
        alert('Failed to create new addon');
    }
}
 </script>

<template>
  <section id="configure" class="theme-dark">
    <div class="container">


      <form class="content-grid" @submit.prevent>
        <section class="card auth-card">
          <div class="card-header">
            <div class="step-indicator">1</div>
            <h3>Authenticate</h3>
          </div>
          <div class="card-body">
            <Authentication
              :stremioAPIBase="stremioAPIBase"
              @auth-key="setAuthKey"
              @auth-success="onAuthSuccess"
            />
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">2</div>
            <h3>Options</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <div class="backup-section">
              <p class="card-desc">Download your current addon configuration before making changes.</p>
              <button
                type="button"
                class="btn accent"
                :disabled="isLocked || addonsCount === 0"
                @click="downloadBackup"
              >
                <span>📥</span>
                <span>Download Backup</span>
              </button>
            </div>
            
            <div class="section-divider"></div>
            
            <div class="option-list">
              <div class="option-row">
                <div class="option-text">
                  <div class="option-title">Remove Cinemeta Search</div>
                  <div class="option-sub">Stops Cinemeta results from appearing when you search.</div>
                </div>
                <label class="switch">
                  <input type="checkbox" v-model="shouldRemoveSearchArtifacts" :disabled="isLocked" />
                  <span class="slider"></span>
                </label>
              </div>

              <div class="option-row">
                <div class="option-text">
                  <div class="option-title">Remove Cinemeta Catalogs</div>
                  <div class="option-sub">Removes Popular, New, and Featured catalogs from your home screen.</div>
                </div>
                <label class="switch">
                  <input type="checkbox" v-model="shouldRemoveStandardCatalogs" :disabled="isLocked" />
                  <span class="slider"></span>
                </label>
              </div>

              <div class="option-row">
                <div class="option-text">
                  <div class="option-title">Remove Cinemeta Metadata</div>
                  <div class="option-sub">Stops Cinemeta from providing metadata for your movies and series.</div>
                </div>
                <label class="switch">
                  <input type="checkbox" v-model="shouldRemoveMetaResource" :disabled="isLocked" />
                  <span class="slider"></span>
                </label>
              </div>
            </div>
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">3</div>
            <h3>Sync Addons</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <p class="card-desc">Apply selected options and sync to Stremio.</p>
            <button
              type="button"
              class="btn primary"
              :disabled="isLocked || isSyncing"
              :aria-busy="isSyncing"
              @click="syncUserAddons"
            >
              <span class="spinner" v-if="isSyncing"></span>
              <span>Sync to Stremio</span>
            </button>
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">⚙️</div>
            <h3>Restore</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <p class="card-desc">Load and restore a previous addon configuration backup.</p>
            
            <div class="restore-controls">
              <button
                type="button"
                class="btn primary"
                :disabled="isLocked"
                @click="loadBackupFile"
              >
                <span>📁</span>
                <span>Load Backup</span>
              </button>
              
              <div class="file-info" v-if="backupFileName">
                <span class="file-label">Loaded:</span>
                <span class="file-name" :class="{ valid: isBackupValid, invalid: !isBackupValid }">
                  {{ backupFileName }}
                </span>
                <span class="file-status" v-if="isBackupValid">✅</span>
                <span class="file-status" v-else>❌</span>
              </div>
            </div>
            
            <button
              type="button"
              class="btn accent restore-btn"
              :disabled="isLocked || !isBackupValid || isRestoring"
              :aria-busy="isRestoring"
              @click="restoreFromBackup"
            >
              <span class="spinner" v-if="isRestoring"></span>
              <span>Restore Configuration</span>
            </button>
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">📦</div>
            <h3>Manage Addons</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <p class="card-desc">Create and manage your custom addon configurations directly.</p>
            
            <div class="management-controls">
              <div class="addon-stats">
                <span class="stats-label">Total Addons:</span>
                <span class="stats-count">{{ addonsCount }}</span>
              </div>
              <button
                type="button"
                class="btn primary"
                :disabled="isLocked"
                @click="toggleAddonManagement"
              >
                <span>{{ isManageAddonVisible ? '📝' : '➕' }}</span>
                <span>{{ isManageAddonVisible ? 'Hide Addon Editor' : 'Create New Addon' }}</span>
              </button>
            </div>
            
            <div class="current-addons-section" v-if="addonsCount > 0">
              <div class="section-divider"></div>
              <h4 class="section-title">Current Addons <span class="drag-hint">💡 Drag to reorder</span></h4>
              <draggable 
                v-model="addons" 
                class="addons-list sortable-list"
                :disabled="isLocked"
                item-key="id"
                handle=".drag-handle"
                animation="150"
                ghost-class="item-ghost"
                chosen-class="item-chosen"
                drag-class="item-dragging"
                :delay="200"
                :delay-on-touch-only="true"
                :touch-start-threshold="10"
                :force-fallback="false"
                scroll-sensitivity="100"
                bubble-scroll="true"
              >
                <template #item="{ element: addon, index }">
                  <AddonItem
                    :key="index"
                    :name="getNestedObjectProperty(addon, 'manifest.name', 'Unknown Addon')"
                    :idx="index"
                    :manifestURL="addon.transportUrl || 'N/A'"
                    :logoURL="getNestedObjectProperty(addon, 'manifest.logo', null)"
                    :isDeletable="addon.manifest?.name !== 'Cinemeta'"
                    :isConfigurable="!!(addon.transportUrl && addon.transportUrl.includes('/manifest.json'))"
                    @delete-addon="removeAddon"
                    @edit-manifest="openEditModal"
                  />
                </template>
              </draggable>
            </div>
            
            <div v-if="isManageAddonVisible" class="addon-management-section">
              <div class="section-divider"></div>
              <h4 class="section-title">Addon Configuration</h4>
              <DynamicForm 
                :manifest="newAddonManifest" 
                @update-manifest="saveNewAddon" 
              />
            </div>
          </div>
        </section>
      </form>
    </div>
  </section>

  <div v-if="isEditModalVisible" class="modal theme-dark" @click.self="closeEditModal">
    <div class="modal-content">
      <div class="modal-header">
        <h3>Edit manifest</h3>
        <button class="btn icon ghost" @click="closeEditModal" aria-label="Close modal">×</button>
      </div>
      <div class="modal-body">
        <DynamicForm :manifest="currentManifest" @update-manifest="saveManifestEdit" />
      </div>
    </div>
  </div>
</template>

<style scoped>
/* VS Code dark-inspired theme tokens */
#configure.theme-dark {
  --bg: #1e1e1e;
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --focus: #3794ff;
  --shadow: rgba(0, 0, 0, 0.35);
}

#configure.theme-dark { color: var(--text); padding: 8px 16px; }

.container { max-width: 960px; margin: 0 auto; }

.page-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 12px;
}

.title-wrap .title {
  margin: 0;
  font-size: 22px;
  font-weight: 600;
  letter-spacing: 0.2px;
}

.subtitle {
  margin: 6px 0 0 0;
  color: var(--muted);
  font-size: 13px;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 10px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 999px;
  font-size: 12px;
  color: var(--text);
}

.content-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 18px;
  width: 100%;
}

/* Cards */
.card {
  width: 100%;
  box-sizing: border-box;
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 10px;
  box-shadow: 0 6px 18px var(--shadow);
  overflow: hidden;
  transition: transform 120ms ease, box-shadow 120ms ease, border-color 120ms ease;
}
.card:hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 24px var(--shadow);
  border-color: #4b4b4b;
}
.card-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  border-bottom: 1px solid var(--border);
  background: linear-gradient(180deg, var(--panel), rgba(0, 0, 0, 0));
}
.card-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
}
.lock-chip {
  margin-left: auto;
  font-size: 12px;
  color: var(--muted);
  background: rgba(255,255,255,0.04);
  border: 1px solid var(--border);
  padding: 4px 8px;
  border-radius: 999px;
}
.step-indicator {
  width: 28px;
  height: 28px;
  border-radius: 6px;
  background: linear-gradient(180deg, var(--accent), var(--accent-2));
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 13px;
  font-weight: 700;
  box-shadow: 0 2px 8px rgba(0, 122, 204, 0.35);
}
.card-body { padding: 16px; }
.card.disabled {
  position: relative;
}
.card.disabled::after {
  content: "";
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, rgba(0,0,0,0.25), rgba(0,0,0,0.35));
  pointer-events: none;
}
.card.disabled .card-body,
.card.disabled .card-header {
  filter: grayscale(0.2) opacity(0.75);
}
.card.disabled .switch .slider {
  background: #2f2f2f;
}
.card.disabled .switch .slider:before {
  background: #a9a9a9;
}
.card-desc {
  color: var(--muted);
  font-size: 13px;
  margin: 0 0 12px 0;
}

/* Vertical stack at all widths; fine-tune spacing on larger screens */
@media (min-width: 720px) {
  #configure.theme-dark { padding: 28px; }
  .content-grid {
    grid-template-columns: 1fr;
    gap: 20px;
  }
  .content-grid > .card { grid-column: auto; }
}
@media (min-width: 1024px) {
  #configure.theme-dark { padding: 32px; }
  .content-grid { gap: 22px; }
}

/* Backup section */
.backup-section {
  margin-bottom: 16px;
}

.backup-section .card-desc {
  margin-bottom: 14px;
}

.section-divider {
  height: 1px;
  background: var(--border);
  margin: 20px 0;
}

/* Restore section */
.restore-controls {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 6px;
  font-size: 13px;
  min-width: 0; /* Allow text to wrap */
}

.file-label {
  color: var(--muted);
  font-weight: 500;
}

.file-name {
  color: var(--text);
  font-family: monospace;
  font-size: 12px;
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-name.valid {
  color: #4ade80; /* Green for valid */
}

.file-name.invalid {
  color: #f87171; /* Red for invalid */
}

.file-status {
  font-size: 14px;
}

.restore-btn {
  width: 100%;
  justify-content: center;
}

@media (max-width: 640px) {
  .restore-controls {
    flex-direction: column;
    align-items: stretch;
  }
  
  .file-info {
    order: 2;
  }
}

/* Addon management section */
.management-controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.addon-stats {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 6px;
  font-size: 13px;
}

.stats-label {
  color: var(--muted);
  font-weight: 500;
}

.stats-count {
  color: var(--accent);
  font-weight: 700;
  font-size: 14px;
}

.current-addons-section {
  margin-top: 16px;
}

.addons-list {
  max-height: 400px;
  overflow-y: auto;
  margin-top: 8px;
}

.addon-management-section {
  margin-top: 16px;
}

.section-title {
  margin: 16px 0 12px 0;
  color: var(--text);
  font-size: 15px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 12px;
}

.drag-hint {
  font-size: 12px;
  color: var(--muted);
  font-weight: 500;
  background: var(--panel-2);
  padding: 4px 8px;
  border-radius: 4px;
  border: 1px solid var(--border);
}

@media (max-width: 640px) {
  .management-controls {
    flex-direction: column;
    align-items: stretch;
  }
  
  .addon-stats {
    order: 1;
    justify-content: center;
  }
}

/* Options list */
.option-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.option-row {
  display: grid;
  grid-template-columns: 1fr auto;
  align-items: center;
  gap: 14px;
  padding: 14px 16px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 8px;
}
.option-title {
  font-size: 14.5px;
  font-weight: 600;
}
.option-sub {
  color: var(--muted);
  font-size: 12.5px;
  margin-top: 4px;
}

/* Toggle switch */
.switch {
  position: relative;
  display: inline-flex;
  align-items: center;
  width: 48px;
  height: 28px;
}
.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}
.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #3a3a3a;
  border: 1px solid var(--border);
  transition: background 120ms ease, border-color 120ms ease;
  border-radius: 999px;
}
.slider:before {
  position: absolute;
  content: "";
  height: 22px;
  width: 22px;
  left: 3px;
  top: 50%;
  transform: translateY(-50%);
  background-color: #cfcfcf;
  border-radius: 50%;
  box-shadow: 0 2px 6px rgba(0,0,0,0.35);
  transition: transform 120ms ease, background 120ms ease;
}
.switch input:focus + .slider {
  outline: 2px solid var(--focus);
  outline-offset: 2px;
}
.switch input:checked + .slider {
  background: linear-gradient(180deg, var(--accent), var(--accent-2));
  border-color: transparent;
}
.switch input:checked + .slider:before {
  transform: translate(18px, -50%);
  background: white;
}

/* Disabled inputs visual consistency */
.switch input:disabled + .slider {
  background: #2f2f2f;
  border-color: #3b3b3b;
}
.switch input:disabled + .slider:before {
  background: #a9a9a9;
}

/* Buttons */
.btn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 11px 16px;
  border-radius: 8px;
  border: 1px solid var(--border);
  background: var(--panel-2);
  color: var(--text);
  font-size: 14.5px;
  line-height: 1;
  cursor: pointer;
  transition: transform 80ms ease, background 120ms ease, border-color 120ms ease, box-shadow 120ms ease;
}
.btn:hover {
  background: #323234;
  border-color: #4b4b4b;
}
.btn:active {
  transform: translateY(1px);
}
.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.btn.primary {
  background: linear-gradient(180deg, #3a3a3a, #2f2f2f);
  border-color: #4a4a4a;
}
.btn.accent {
  background: linear-gradient(180deg, var(--accent), var(--accent-2));
  border-color: transparent;
  color: white;
}
.btn.icon {
  padding: 8px 12px;
  border-radius: 6px;
}
.btn.ghost {
  background: transparent;
  border-color: var(--border);
}

/* Spinner */
.spinner {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.25);
  border-top-color: rgba(255, 255, 255, 0.95);
  animation: spin 800ms linear infinite;
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Modal */
.modal {
  position: fixed;
  inset: 0;
  z-index: 1000;
  background: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  overflow: auto;
  backdrop-filter: blur(2px);
}
.modal.theme-dark {
  --bg: #1e1e1e;
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --focus: #3794ff;
  --shadow: rgba(0, 0, 0, 0.35);
}
.modal-content {
  width: min(900px, 95vw);
  max-width: calc(100vw - 32px);
  max-height: calc(100vh - 40px);
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 10px;
  box-shadow: 0 12px 28px var(--shadow);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 18px 20px;
  border-bottom: 1px solid var(--border);
  background: var(--panel);
  color: var(--text);
  flex-shrink: 0;
}
.modal-header h3 {
  margin: 0;
  font-size: 17px;
  font-weight: 600;
  color: var(--text);
  line-height: 1.3;
}
.modal-header .btn {
  background: var(--panel-2);
  border: 1px solid var(--border);
  color: var(--text);
  min-width: 36px;
  height: 36px;
  font-size: 18px;
  transition: all 120ms ease;
}
.modal-header .btn:hover {
  background: #323234;
  border-color: #4b4b4b;
  transform: scale(1.05);
}
.modal-body {
  padding: 20px;
  overflow: auto;
  background: var(--panel);
  flex: 1;
  /* Ensure the modal body inherits theme variables for nested components */
  --panel: #252526;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --focus: #3794ff;
}

/* Custom scrollbar for modal body */
.modal-body::-webkit-scrollbar {
  width: 8px;
}
.modal-body::-webkit-scrollbar-track {
  background: var(--panel-2);
  border-radius: 4px;
}
.modal-body::-webkit-scrollbar-thumb {
  background: var(--border);
  border-radius: 4px;
}
.modal-body::-webkit-scrollbar-thumb:hover {
  background: #4a4a4a;
}

/* Drag and Drop Styling */
.sortable-list {
  position: relative;
}

.item-ghost {
  opacity: 0.5;
  background: var(--panel-2);
  border: 2px dashed var(--accent);
  border-radius: 10px;
}

.item-chosen {
  transform: scale(1.02);
  box-shadow: 0 8px 20px rgba(0, 122, 204, 0.25);
  border-color: var(--accent);
}

.item-dragging {
  opacity: 0.8;
  transform: rotate(2deg);
  cursor: grabbing;
}

.item-dragging :where(.details, .actions) {
  opacity: 0.7;
}

/* Mobile modal optimizations */
@media (max-width: 768px) {
  .modal {
    padding: 12px;
    align-items: flex-start;
    padding-top: 20px;
  }
  
  .modal-content {
    width: 100%;
    max-width: none;
    max-height: calc(100vh - 32px);
    border-radius: 8px;
  }
  
  .modal-header {
    padding: 16px;
    gap: 12px;
  }
  
  .modal-header h3 {
    font-size: 16px;
  }
  
  .modal-body {
    padding: 16px;
  }
}

@media (max-width: 480px) {
  .modal {
    padding: 8px;
    padding-top: 16px;
  }
  
  .modal-header {
    padding: 14px;
  }
  
  .modal-body {
    padding: 14px;
  }
}

/* Reduce motion */
@media (prefers-reduced-motion: reduce) {
  * {
    transition: none !important;
    animation: none !important;
  }
}
</style>

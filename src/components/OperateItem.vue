<template>
  <!-- operate item -->
  <el-form class="connection-form" size="mini">
    <el-form-item>
      <el-row :gutter="6">
        <!-- db index select -->
        <el-col :span="12">
          <el-select class="db-select" v-model="selectedDbIndex" placeholder="DB" @change="changeDb()" :filter-method="filterDbCustomName" @visible-change="revertDbFilter" filterable default-first-option>
            <el-option
              v-for="index in dbsCopy"
              :key="index"
              :label="`DB${index}`"
              :value="index">
              <span>
                {{`DB${index}`}}
                <span class="db-select-key-count" v-if="dbKeysCount[index]">[{{dbKeysCount[index]}}]</span>
                <span class="db-select-custom-name">
                  <span class="db-select-key-count">{{dbNames[index]}}</span>

                  <span class="el-icon-edit-outline" @click.stop.prevent='customDbName(index)'></span>
                </span>
              </span>
            </el-option>
            <!-- <span slot="prefix" class="fa fa-sitemap" style="font-size: 80%"></span> -->
          </el-select>
        </el-col>

        <!-- new key btn -->
        <el-col :span="12">
          <el-button class="new-key-btn" @click="newKeyDialog=true">
            <i class="el-icon-plus"></i>
            {{ $t('message.add_new_key') }}
          </el-button>
        </el-col>
      </el-row>
    </el-form-item>

    <!-- search match -->
    <!-- <el-form-item class="search-item">
      <el-row>
        <el-col :span="24">
          <el-input class="search-input" v-model="searchMatch" @keyup.enter.native="changeMatchMode()" :placeholder="$t('message.enter_to_search')" size="mini">
            <span slot="suffix">
              <i class="el-input__icon search-icon" :class="searchIcon"  @click="changeMatchMode()"></i>

              <el-tooltip effect="dark" :content="$t('message.exact_search')" placement="bottom">
                <el-checkbox v-model="searchExact"></el-checkbox>
              </el-tooltip>
            </span>
          </el-input>
        </el-col>
      </el-row>
    </el-form-item> -->

    <!-- autocomplete search input -->
    <el-form-item class="search-item">
      <el-autocomplete
        class="search-input"
        v-model="searchMatch"
        @select="changeMatchMode"
        @keyup.enter="changeMatchMode()"
        :debounce="searchDebounce"
        :fetch-suggestions="querySearch"
        :placeholder="$t('message.enter_to_search')"
        :trigger-on-focus="false"
        :select-when-unmatched='true'>
        <template slot="suffix">
          <!-- cancel search -->
          <i v-if="(searchIcon=='el-icon-loading') && showCancelIcon" class="el-input__icon search-icon el-icon-error" @click="cancelSearch()" :title="$t('el.messagebox.cancel')"></i>
          <!-- start search -->
          <i class="el-input__icon search-icon" :class="searchIcon"  @click="changeMatchMode()"></i>

          <!-- extract search -->
          <el-tooltip effect="dark" :content="$t('message.exact_search')" placement="bottom">
            <el-checkbox v-model="searchExact"></el-checkbox>
          </el-tooltip>
        </template>
      </el-autocomplete>
    </el-form-item>

    <!-- new key dialog -->
    <el-dialog :title="$t('message.add_new_key')" :visible.sync="newKeyDialog" :close-on-click-modal='false' @opened="openNewKeyDialog" append-to-body>
      <el-form label-position="top" size="mini">
        <el-form-item :label="$t('message.key_name')">
          <el-input v-model='newKeyName' ref="newKeyNameInput"></el-input>
        </el-form-item>

        <el-form-item :label="$t('message.key_type')">
          <el-select style='width: 100%' v-model="selectedNewKeyType">
              <el-option
                v-for="(type, showType) in newKeyTypes"
                :key="type"
                :label="showType"
                :value="type">
              </el-option>
            </el-select>
        </el-form-item>
      </el-form>

      <div slot="footer" class="dialog-footer">
        <el-button @click="newKeyDialog = false">{{ $t('el.messagebox.cancel') }}</el-button>
        <el-button type="primary" @click="addNewKey">{{ $t('el.messagebox.confirm') }}</el-button>
      </div>
    </el-dialog>
  </el-form>
</template>

<script type="text/javascript">
export default {
  data() {
    return {
      dbs: [0],
      dbsCopy: [0],
      selectedDbIndex: 0,
      searchMatch: '',
      searchExact: false,
      searchIcon: 'el-icon-search',
      searchHistory: new Set(),
      searchDebounce: 100,
      newKeyDialog: false,
      newKeyName: '',
      selectedNewKeyType: 'string',
      newKeyTypes: {
        String: 'string',
        Hash: 'hash',
        List: 'list',
        Set: 'set',
        Zset: 'zset',
        Stream: 'stream',
        ReJSON: 'rejson',
      },
      dbKeysCount: {},
      dbNames: {},
      showCancelIcon: false,
      rmCancelIconTimer: null,
    };
  },
  props: ['client', 'config'],
  watch: {
    dbs(newValue, oldValue) {
      this.dbsCopy = newValue.concat();
    },
  },
  created() {
    this.$bus.$on('changeDb', (client, dbIndex) => {
      if (!this.client || client.options.connectionName != this.client.options.connectionName) {
        return;
      }

      if (this.client.condition.select == dbIndex) {
        return;
      }

      this.changeDb(dbIndex);
    });
    this.$bus.$on('changeMatchMode', (client, pattern) => {
      if (client !== this.client) {
        return;
      }

      this.searchMatch = pattern;
      this.changeMatchMode();
    });
  },
  methods: {
    initShow() {
      this.initDatabaseSelect();
      this.initCustomDbName();
    },
    setDb(db) {
      this.selectedDbIndex = db;
    },
    initDatabaseSelect() {
      this.client.config('get', 'databases').then((reply) => {
        this.dbs = [...Array(parseInt(reply[1])).keys()];
        this.getDatabasesFromInfo();
      }).catch((e) => {
        // config command may be renamed
        this.dbs = [...Array(16).keys()];
        // read dbs from info
        this.getDatabasesFromInfo(true);
      });
    },
    initCustomDbName() {
      const dbKey = this.$storage.getStorageKeyByName('custom_db', this.config.connectionName);
      const customNames = JSON.parse(localStorage.getItem(dbKey));

      if (customNames) {
        this.dbNames = customNames;
      }
    },
    getDatabasesFromInfo(guessMaxDb = false) {
      if (!this.client) {
        return;
      }

      this.dbKeysCount = {};
      this.client.info().then((info) => {
        const keyspace = info.split('# Keyspace')[1].trim().split('\n');
        let keyCount = [];

        for (const line of keyspace) {
          keyCount = line.match(/db(\d+)\:keys=(\d+)/);
          keyCount && this.$set(this.dbKeysCount, keyCount[1], keyCount[2]);
        }

        if (!guessMaxDb || !keyCount || !keyCount[1]) {
          return;
        }
        // max/last db which exists keys
        const maxDb = parseInt(keyCount[1]);

        if (maxDb > 16) {
          this.dbs = [...Array(maxDb + 1).keys()];
        }
      }).catch(() => {});
    },
    resetStatus() {
      this.dbs = [0];
      // this.selectedDbIndex = 0;
      this.searchMatch = '';
      this.searchExact = false;
    },
    changeDb(dbIndex = false) {
      if (dbIndex !== false) {
        this.selectedDbIndex = parseInt(dbIndex);
      }

      this.client.select(this.selectedDbIndex)
        .then(() => {
        // clear the search input
          this.searchMatch = '';
          this.$parent.$parent.$parent.$refs.keyList.refreshKeyList();
          const dbKey = this.$storage.getStorageKeyByName('last_db', this.config.connectionName);
          // store the last selected db
          localStorage.setItem(dbKey, this.selectedDbIndex);
          // tell cli to change db
          this.client.options.db = this.selectedDbIndex;
          this.$bus.$emit('changeDb', this.client, this.selectedDbIndex);
        })
      // select is not allowed in cluster mode
        .catch((e) => {
          this.$message.error({
            message: e.message,
            duration: 3000,
          });

          // reset to db0
          this.selectedDbIndex = 0;
        });
    },
    customDbName(db) {
      const name = this.dbNames[db];

      this.$prompt(this.$t('message.custom_name'), { inputValue: name }).then(({ value }) => {
        this.$set(this.dbNames, db, value);
        const dbKey = this.$storage.getStorageKeyByName('custom_db', this.config.connectionName);
        localStorage.setItem(dbKey, JSON.stringify(this.dbNames));
      }).catch(() => {});
    },
    filterDbCustomName(query) {
      query = query.toLocaleLowerCase();

      this.dbsCopy = this.dbs.filter(dbIndex => {
        if (`db${dbIndex}`.includes(query)) {
          return true;
        }

        const dbName = this.dbNames[dbIndex];
        if (dbName && dbName.toLowerCase().includes(query)) {
          return true;
        }

        return false;
      });
    },
    openNewKeyDialog() {
      this.$refs.newKeyNameInput.focus();
      this.$refs.newKeyNameInput.select();
    },
    addNewKey() {
      if (!this.newKeyName) {
        return;
      }

      // key to buffer
      const key = Buffer.from(this.newKeyName);

      const promise = this.setDefaultValue(key, this.selectedNewKeyType);

      promise.then(() => {
        this.$bus.$emit('refreshKeyList', this.client, key, 'add');
        this.$bus.$emit('clickedKey', this.client, key, true);
      }).catch((e) => {
        this.$message.error(e.message);
      });

      this.newKeyDialog = false;
    },
    setDefaultValue(key, type) {
      switch (type) {
        case 'string': {
          return this.client.set(key, '');
        }
        case 'hash': {
          return this.client.hset(key, 'New field', 'New value');
        }
        case 'list': {
          return this.client.lpush(key, 'New member');
        }
        case 'set': {
          return this.client.sadd(key, 'New member');
        }
        case 'zset': {
          return this.client.zadd(key, 0, 'New member');
        }
        case 'stream': {
          return this.client.xadd(key, '*', 'New key', 'New value');
        }
        case 'rejson': {
          return this.client.call('JSON.SET', [key, '$', '{"New key":"New value"}']);
        }
      }
    },
    toggleCancelIcon(show = false) {
      this.showCancelIcon = false;
      clearTimeout(this.rmCancelIconTimer);

      if (show) {
        this.rmCancelIconTimer = setTimeout(() => {
          this.showCancelIcon = true;
        }, 800);
      }
    },
    changeMatchMode() {
      // already searching status
      // if (this.searchIcon == 'el-icon-loading') {
      //   return false;
      // }

      // prevent enter too fast but show candidate query
      setTimeout(() => {
        this.searchHistory.add(this.searchMatch);
      }, this.searchDebounce + 100);

      this.$parent.$parent.$parent.$refs.keyList.refreshKeyList();
    },
    querySearch(input, cb) {
      const items = [];

      if (!this.searchHistory.size) {
        return cb([]);
      }

      this.searchHistory.forEach((value) => {
        if (value.toLowerCase().indexOf(input.toLowerCase()) !== -1) {
          items.push({ value });
        }
      });

      cb(items);
    },
    initHistory() {
      const key = this.$storage.getStorageKeyByName('search_tip', this.config.name);
      const tips = localStorage.getItem(key);

      const suggestions = tips ? JSON.parse(tips) : [];
      this.searchHistory = new Set(suggestions);

      // record raw tips count
      this.searchHistoryCount = this.searchHistory.size;

      ipcRenderer.on('closingWindow', (event, arg) => {
        this.storeHistory();
      });
    },
    storeHistory() {
      // not changed
      if (this.searchHistory.size === this.searchHistoryCount) {
        return;
      }
      const key = this.$storage.getStorageKeyByName('search_tip', this.config.name);
      localStorage.setItem(key, JSON.stringify(Array.from(this.searchHistory).slice(-200)));
    },
    cancelSearch() {
      // stop scanning in keyList
      this.$parent.$parent.$parent.$refs.keyList.cancelScanning();
      // reset search status
      this.$parent.$parent.$parent.$refs.keyList.resetSearchStatus();
    },
    revertDbFilter(visible) {
      // revert only when select shows
      visible && (this.dbsCopy = this.dbs.concat());
    },
  },
  mounted() {
    this.initHistory();
  },
};
</script>

<style type="text/css">
  .connection-menu .db-select {
    width: 100%;
  }
  .el-select-dropdown__item .db-select-key-count {
    color: #a9a9ab;
    font-size: 82%;
    vertical-align: top;
  }
  .el-select-dropdown__item .db-select-custom-name {
    float: right;
    margin-left: 4px;
  }

  /*fix el-select height different from el-input*/
  .connection-menu .db-select .el-input__inner, .connection-menu .new-key-btn {
    /*margin-top: 0.5px;*/
    height: 28px;
  }
  .connection-menu .new-key-btn {
    width: 100%;
  }
  .connection-menu .search-item {
    margin-top: -10px;
    margin-bottom: 15px;
  }
  /*fix extract checkbox height*/
  .connection-menu .search-item .el-input__suffix-inner {
    display: inline-block;
  }
  .connection-menu .search-input {
    width: 100%;
  }
  .connection-menu .search-input .el-input__inner {
    padding-right: 62px;
    /*margin-top: -10px;;
    margin-bottom: 15px;*/
  }

  .connection-menu .el-submenu__title .el-submenu__icon-arrow {
    right: 7px;
    top: 54%;
  }
  .connection-menu .el-submenu [class^=el-icon-] {
    font-size: 12px;
    margin: 0px;
    width: auto;
    /*color: grey;*/
    vertical-align: baseline;
  }

  .connection-menu .el-submenu.is-opened {
    /*background: #ECF5FF;*/
  }

  .connection-menu .connection-form {
    /*padding-right: 8px;*/
  }

  .connection-menu .search-item .search-icon {
    font-size: 128%;
    color: #a5a8ad;
    cursor: pointer;
    width: 20px;
  }
  .connection-menu .search-item .el-checkbox__input {
    /*line-height: 28px;*/
    display: inline;
  }
</style>

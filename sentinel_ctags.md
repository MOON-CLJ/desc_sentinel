ctags -nux src/sentinel.c

- [ ] REDIS_SENTINEL_PORT macro        43 src/sentinel.c   #define REDIS_SENTINEL_PORT 26379

- [ ] sentinelAddr     struct       48 src/sentinel.c   typedef struct sentinelAddr {

- [ ] ip               member       49 src/sentinel.c   char *ip;

- [ ] port             member       50 src/sentinel.c   int port;

- [ ] sentinelAddr     typedef      51 src/sentinel.c   } sentinelAddr;

- [ ] SRI_MASTER       macro        54 src/sentinel.c   #define SRI_MASTER (1<<0)

- [ ] SRI_SLAVE        macro        55 src/sentinel.c   #define SRI_SLAVE (1<<1)

- [ ] SRI_SENTINEL     macro        56 src/sentinel.c   #define SRI_SENTINEL (1<<2)

- [ ] SRI_DISCONNECTED macro        57 src/sentinel.c   #define SRI_DISCONNECTED (1<<3)

- [ ] SRI_S_DOWN       macro        58 src/sentinel.c   #define SRI_S_DOWN (1<<4) /* Subjectively down (no quorum). */

- [ ] SRI_O_DOWN       macro        59 src/sentinel.c   #define SRI_O_DOWN (1<<5) /* Objectively down (confirmed by others). */

- [ ] SRI_MASTER_DOWN  macro        60 src/sentinel.c   #define SRI_MASTER_DOWN (1<<6) /* A Sentinel with this flag set thinks that

- [ ] SRI_FAILOVER_IN_PROGRESS macro        62 src/sentinel.c   #define SRI_FAILOVER_IN_PROGRESS (1<<7) /* Failover is in progress for

- [ ] SRI_PROMOTED     macro        64 src/sentinel.c   #define SRI_PROMOTED (1<<8) /* Slave selected for promotion. */

- [ ] SRI_RECONF_SENT  macro        65 src/sentinel.c   #define SRI_RECONF_SENT (1<<9) /* SLAVEOF <newmaster> sent. */

- [ ] SRI_RECONF_INPROG macro        66 src/sentinel.c   #define SRI_RECONF_INPROG (1<<10) /* Slave synchronization in progress. */

- [ ] SRI_RECONF_DONE  macro        67 src/sentinel.c   #define SRI_RECONF_DONE (1<<11) /* Slave synchronized with new master. */

- [ ] SRI_FORCE_FAILOVER macro        68 src/sentinel.c   #define SRI_FORCE_FAILOVER (1<<12) /* Force failover with master up. */

- [ ] SRI_SCRIPT_KILL_SENT macro        69 src/sentinel.c   #define SRI_SCRIPT_KILL_SENT (1<<13) /* SCRIPT KILL already sent on -BUSY */

- [ ] SENTINEL_INFO_PERIOD macro        72 src/sentinel.c   #define SENTINEL_INFO_PERIOD 10000

- [ ] SENTINEL_PING_PERIOD macro        73 src/sentinel.c   #define SENTINEL_PING_PERIOD 1000

- [ ] SENTINEL_ASK_PERIOD macro        74 src/sentinel.c   #define SENTINEL_ASK_PERIOD 1000

- [ ] SENTINEL_PUBLISH_PERIOD macro        75 src/sentinel.c   #define SENTINEL_PUBLISH_PERIOD 2000

- [ ] SENTINEL_DEFAULT_DOWN_AFTER macro        76 src/sentinel.c   #define SENTINEL_DEFAULT_DOWN_AFTER 30000

- [ ] SENTINEL_HELLO_CHANNEL macro        77 src/sentinel.c   #define SENTINEL_HELLO_CHANNEL "__sentinel__:hello"

- [ ] SENTINEL_TILT_TRIGGER macro        78 src/sentinel.c   #define SENTINEL_TILT_TRIGGER 2000

- [ ] SENTINEL_TILT_PERIOD macro        79 src/sentinel.c   #define SENTINEL_TILT_PERIOD (SENTINEL_PING_PERIOD*30)

- [ ] SENTINEL_DEFAULT_SLAVE_PRIORITY macro        80 src/sentinel.c   #define SENTINEL_DEFAULT_SLAVE_PRIORITY 100

- [ ] SENTINEL_SLAVE_RECONF_TIMEOUT macro        81 src/sentinel.c   #define SENTINEL_SLAVE_RECONF_TIMEOUT 10000

- [ ] SENTINEL_DEFAULT_PARALLEL_SYNCS macro        82 src/sentinel.c   #define SENTINEL_DEFAULT_PARALLEL_SYNCS 1

- [ ] SENTINEL_MIN_LINK_RECONNECT_PERIOD macro        83 src/sentinel.c   #define SENTINEL_MIN_LINK_RECONNECT_PERIOD 15000

- [ ] SENTINEL_DEFAULT_FAILOVER_TIMEOUT macro        84 src/sentinel.c   #define SENTINEL_DEFAULT_FAILOVER_TIMEOUT (60*3*1000)

- [ ] SENTINEL_MAX_PENDING_COMMANDS macro        85 src/sentinel.c   #define SENTINEL_MAX_PENDING_COMMANDS 100

- [ ] SENTINEL_ELECTION_TIMEOUT macro        86 src/sentinel.c   #define SENTINEL_ELECTION_TIMEOUT 10000

- [ ] SENTINEL_MAX_DESYNC macro        87 src/sentinel.c   #define SENTINEL_MAX_DESYNC 1000

- [ ] SENTINEL_FAILOVER_STATE_NONE macro        90 src/sentinel.c   #define SENTINEL_FAILOVER_STATE_NONE 0 /* No failover in progress. */

- [ ] SENTINEL_FAILOVER_STATE_WAIT_START macro        91 src/sentinel.c   #define SENTINEL_FAILOVER_STATE_WAIT_START 1 /* Wait for failover_start_time*/

- [ ] SENTINEL_FAILOVER_STATE_SELECT_SLAVE macro        92 src/sentinel.c   #define SENTINEL_FAILOVER_STATE_SELECT_SLAVE 2 /* Select slave to promote */

- [ ] SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE macro        93 src/sentinel.c   #define SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE 3 /* Slave -> Master */

- [ ] SENTINEL_FAILOVER_STATE_WAIT_PROMOTION macro        94 src/sentinel.c   #define SENTINEL_FAILOVER_STATE_WAIT_PROMOTION 4 /* Wait slave to change role */

- [ ] SENTINEL_FAILOVER_STATE_RECONF_SLAVES macro        95 src/sentinel.c   #define SENTINEL_FAILOVER_STATE_RECONF_SLAVES 5 /* SLAVEOF newmaster */

- [ ] SENTINEL_FAILOVER_STATE_UPDATE_CONFIG macro        96 src/sentinel.c   #define SENTINEL_FAILOVER_STATE_UPDATE_CONFIG 6 /* Monitor promoted slave. */

- [ ] SENTINEL_MASTER_LINK_STATUS_UP macro        98 src/sentinel.c   #define SENTINEL_MASTER_LINK_STATUS_UP 0

- [ ] SENTINEL_MASTER_LINK_STATUS_DOWN macro        99 src/sentinel.c   #define SENTINEL_MASTER_LINK_STATUS_DOWN 1

- [ ] SENTINEL_NO_FLAGS macro       104 src/sentinel.c   #define SENTINEL_NO_FLAGS 0

- [ ] SENTINEL_GENERATE_EVENT macro       105 src/sentinel.c   #define SENTINEL_GENERATE_EVENT (1<<16)

- [ ] SENTINEL_LEADER  macro       106 src/sentinel.c   #define SENTINEL_LEADER (1<<17)

- [ ] SENTINEL_OBSERVER macro       107 src/sentinel.c   #define SENTINEL_OBSERVER (1<<18)

- [ ] SENTINEL_SCRIPT_NONE macro       110 src/sentinel.c   #define SENTINEL_SCRIPT_NONE 0

- [ ] SENTINEL_SCRIPT_RUNNING macro       111 src/sentinel.c   #define SENTINEL_SCRIPT_RUNNING 1

- [ ] SENTINEL_SCRIPT_MAX_QUEUE macro       112 src/sentinel.c   #define SENTINEL_SCRIPT_MAX_QUEUE 256

- [ ] SENTINEL_SCRIPT_MAX_RUNNING macro       113 src/sentinel.c   #define SENTINEL_SCRIPT_MAX_RUNNING 16

- [ ] SENTINEL_SCRIPT_MAX_RUNTIME macro       114 src/sentinel.c   #define SENTINEL_SCRIPT_MAX_RUNTIME 60000 /* 60 seconds max exec time. */

- [ ] SENTINEL_SCRIPT_MAX_RETRY macro       115 src/sentinel.c   #define SENTINEL_SCRIPT_MAX_RETRY 10

- [ ] SENTINEL_SCRIPT_RETRY_DELAY macro       116 src/sentinel.c   #define SENTINEL_SCRIPT_RETRY_DELAY 30000 /* 30 seconds between retries. */

- [ ] sentinelRedisInstance struct      118 src/sentinel.c   typedef struct sentinelRedisInstance {

- [ ] flags            member      119 src/sentinel.c   int flags; /* See SRI_... defines */

- [ ] name             member      120 src/sentinel.c   char *name; /* Master name from the point of view of this sentinel. */

- [ ] runid            member      121 src/sentinel.c   char *runid; /* run ID of this instance. */

- [ ] config_epoch     member      122 src/sentinel.c   uint64_t config_epoch; /* Configuration epoch. */

- [ ] addr             member      123 src/sentinel.c   sentinelAddr *addr; /* Master host. */

- [ ] cc               member      124 src/sentinel.c   redisAsyncContext *cc; /* Hiredis context for commands. */

- [ ] pc               member      125 src/sentinel.c   redisAsyncContext *pc; /* Hiredis context for Pub / Sub. */

- [ ] pending_commands member      126 src/sentinel.c   int pending_commands; /* Number of commands sent waiting for a reply. */

- [ ] cc_conn_time     member      127 src/sentinel.c   mstime_t cc_conn_time; /* cc connection time. */

- [ ] pc_conn_time     member      128 src/sentinel.c   mstime_t pc_conn_time; /* pc connection time. */

- [ ] pc_last_activity member      129 src/sentinel.c   mstime_t pc_last_activity; /* Last time we received any message. */

- [ ] last_avail_time  member      130 src/sentinel.c   mstime_t last_avail_time; /* Last time the instance replied to ping with

- [ ] last_ping_time   member      132 src/sentinel.c   mstime_t last_ping_time; /* Last time a pending ping was sent in the

- [ ] last_pong_time   member      136 src/sentinel.c   mstime_t last_pong_time; /* Last time the instance replied to ping,

- [ ] last_pub_time    member      139 src/sentinel.c   mstime_t last_pub_time; /* Last time we sent hello via Pub/Sub. */

- [ ] last_hello_time  member      140 src/sentinel.c   mstime_t last_hello_time; /* Only used if SRI_SENTINEL is set. Last time

- [ ] last_master_down_reply_time member      143 src/sentinel.c   mstime_t last_master_down_reply_time; /* Time of last reply to

- [ ] s_down_since_time member      145 src/sentinel.c   mstime_t s_down_since_time; /* Subjectively down since time. */

- [ ] o_down_since_time member      146 src/sentinel.c   mstime_t o_down_since_time; /* Objectively down since time. */

- [ ] down_after_period member      147 src/sentinel.c   mstime_t down_after_period; /* Consider it down after that period. */

- [ ] info_refresh     member      148 src/sentinel.c   mstime_t info_refresh; /* Time at which we received INFO output from it. */

- [ ] role_reported    member      155 src/sentinel.c   int role_reported;

- [ ] role_reported_time member      156 src/sentinel.c   mstime_t role_reported_time;

- [ ] slave_conf_change_time member      157 src/sentinel.c   mstime_t slave_conf_change_time; /* Last time slave master addr changed. */

- [ ] sentinels        member      160 src/sentinel.c   dict *sentinels; /* Other sentinels monitoring the same master. */

- [ ] slaves           member      161 src/sentinel.c   dict *slaves; /* Slaves for this master instance. */

- [ ] quorum           member      162 src/sentinel.c   unsigned int quorum;/* Number of sentinels that need to agree on failure. */

- [ ] parallel_syncs   member      163 src/sentinel.c   int parallel_syncs; /* How many slaves to reconfigure at same time. */

- [ ] auth_pass        member      164 src/sentinel.c   char *auth_pass; /* Password to use for AUTH against master & slaves. */

- [ ] master_link_down_time member      167 src/sentinel.c   mstime_t master_link_down_time; /* Slave replication link down time. */

- [ ] slave_priority   member      168 src/sentinel.c   int slave_priority; /* Slave priority according to its INFO output. */

- [ ] slave_reconf_sent_time member      169 src/sentinel.c   mstime_t slave_reconf_sent_time; /* Time at which we sent SLAVE OF <new> */

- [ ] master           member      170 src/sentinel.c   struct sentinelRedisInstance *master; /* Master instance if it's slave. */

- [ ] slave_master_host member      171 src/sentinel.c   char *slave_master_host; /* Master host as reported by INFO */

- [ ] slave_master_port member      172 src/sentinel.c   int slave_master_port; /* Master port as reported by INFO */

- [ ] slave_master_link_status member      173 src/sentinel.c   int slave_master_link_status; /* Master link status as reported by INFO */

- [ ] slave_repl_offset member      174 src/sentinel.c   unsigned long long slave_repl_offset; /* Slave replication offset. */

- [ ] leader           member      176 src/sentinel.c   char *leader; /* If this is a master instance, this is the runid of

- [ ] leader_epoch     member      180 src/sentinel.c   uint64_t leader_epoch; /* Epoch of the 'leader' field. */

- [ ] failover_epoch   member      181 src/sentinel.c   uint64_t failover_epoch; /* Epoch of the currently started failover. */

- [ ] failover_state   member      182 src/sentinel.c   int failover_state; /* See SENTINEL_FAILOVER_STATE_* defines. */

- [ ] failover_state_change_time member      183 src/sentinel.c   mstime_t failover_state_change_time;

- [ ] failover_start_time member      184 src/sentinel.c   mstime_t failover_start_time; /* Last failover attempt start time. */

- [ ] failover_timeout member      185 src/sentinel.c   mstime_t failover_timeout; /* Max time to refresh failover state. */

- [ ] failover_delay_logged member      186 src/sentinel.c   mstime_t failover_delay_logged; /* For what failover_start_time value we

- [ ] promoted_slave   member      188 src/sentinel.c   struct sentinelRedisInstance *promoted_slave; /* Promoted slave instance. */

- [ ] notification_script member      191 src/sentinel.c   char *notification_script;

- [ ] client_reconfig_script member      192 src/sentinel.c   char *client_reconfig_script;

- [ ] sentinelRedisInstance typedef     193 src/sentinel.c   } sentinelRedisInstance;

- [ ] sentinelState    struct      196 src/sentinel.c   struct sentinelState {

- [ ] current_epoch    member      197 src/sentinel.c   uint64_t current_epoch; /* Current epoch. */

- [ ] masters          member      198 src/sentinel.c   dict *masters; /* Dictionary of master sentinelRedisInstances.

- [ ] tilt             member      201 src/sentinel.c   int tilt; /* Are we in TILT mode? */

- [ ] running_scripts  member      202 src/sentinel.c   int running_scripts; /* Number of scripts in execution right now. */

- [ ] tilt_start_time  member      203 src/sentinel.c   mstime_t tilt_start_time; /* When TITL started. */

- [ ] previous_time    member      204 src/sentinel.c   mstime_t previous_time; /* Last time we ran the time handler. */

- [ ] scripts_queue    member      205 src/sentinel.c   list *scripts_queue; /* Queue of user scripts to execute. */

- [ ] announce_ip      member      206 src/sentinel.c   char *announce_ip; /* IP addr that is gossiped to other sentinels if

- [ ] announce_port    member      208 src/sentinel.c   int announce_port; /* Port that is gossiped to other sentinels if

- [ ] sentinel         variable    210 src/sentinel.c   } sentinel;

- [ ] sentinelScriptJob struct      213 src/sentinel.c   typedef struct sentinelScriptJob {

- [ ] flags            member      214 src/sentinel.c   int flags; /* Script job flags: SENTINEL_SCRIPT_* */

- [ ] retry_num        member      215 src/sentinel.c   int retry_num; /* Number of times we tried to execute it. */

- [ ] argv             member      216 src/sentinel.c   char **argv; /* Arguments to call the script. */

- [ ] start_time       member      217 src/sentinel.c   mstime_t start_time; /* Script execution time if the script is running,

- [ ] pid              member      222 src/sentinel.c   pid_t pid; /* Script execution pid. */

- [ ] sentinelScriptJob typedef     223 src/sentinel.c   } sentinelScriptJob;

- [ ] redisAeEvents    struct      230 src/sentinel.c   typedef struct redisAeEvents {

- [ ] context          member      231 src/sentinel.c   redisAsyncContext *context;

- [ ] loop             member      232 src/sentinel.c   aeEventLoop *loop;

- [ ] fd               member      233 src/sentinel.c   int fd;

- [ ] reading          member      234 src/sentinel.c   int reading, writing;

- [ ] writing          member      234 src/sentinel.c   int reading, writing;

- [ ] redisAeEvents    typedef     235 src/sentinel.c   } redisAeEvents;

- [ ] redisAeReadEvent function    237 src/sentinel.c   static void redisAeReadEvent(aeEventLoop *el, int fd, void *privdata, int mask) {

- [ ] redisAeWriteEvent function    244 src/sentinel.c   static void redisAeWriteEvent(aeEventLoop *el, int fd, void *privdata, int mask) {

- [ ] redisAeAddRead   function    251 src/sentinel.c   static void redisAeAddRead(void *privdata) {

- [ ] redisAeDelRead   function    260 src/sentinel.c   static void redisAeDelRead(void *privdata) {

- [ ] redisAeAddWrite  function    269 src/sentinel.c   static void redisAeAddWrite(void *privdata) {

- [ ] redisAeDelWrite  function    278 src/sentinel.c   static void redisAeDelWrite(void *privdata) {

- [ ] redisAeCleanup   function    287 src/sentinel.c   static void redisAeCleanup(void *privdata) {

- [ ] redisAeAttach    function    294 src/sentinel.c   static int redisAeAttach(aeEventLoop *loop, redisAsyncContext *ac) {

- [ ] dictInstancesValDestructor function    351 src/sentinel.c   void dictInstancesValDestructor (void *privdata, void *obj) {

- [ ] instancesDictType variable    360 src/sentinel.c   dictType instancesDictType = {

- [ ] leaderVotesDictType variable    373 src/sentinel.c   dictType leaderVotesDictType = {

- [ ] sentinelcmds     variable    390 src/sentinel.c   struct redisCommand sentinelcmds[] = {

- [ ] initSentinelConfig function    405 src/sentinel.c   void initSentinelConfig(void) {

- [ ] initSentinel     function    410 src/sentinel.c   void initSentinel(void) {

- [ ] sentinelIsRunning function    438 src/sentinel.c   void sentinelIsRunning(void) {

- [ ] createSentinelAddr function    464 src/sentinel.c   sentinelAddr *createSentinelAddr(char *hostname, int port) {

- [ ] dupSentinelAddr  function    483 src/sentinel.c   sentinelAddr *dupSentinelAddr(sentinelAddr *src) {

- [ ] releaseSentinelAddr function    493 src/sentinel.c   void releaseSentinelAddr(sentinelAddr *sa) {

- [ ] sentinelAddrIsEqual function    499 src/sentinel.c   int sentinelAddrIsEqual(sentinelAddr *a, sentinelAddr *b) {

- [ ] sentinelEvent    function    529 src/sentinel.c   void sentinelEvent(int level, char *type, sentinelRedisInstance *ri,

- [ ] sentinelGenerateInitialMonitorEvents function    590 src/sentinel.c   void sentinelGenerateInitialMonitorEvents(void) {

- [ ] sentinelReleaseScriptJob function    605 src/sentinel.c   void sentinelReleaseScriptJob(sentinelScriptJob *sj) {

- [ ] SENTINEL_SCRIPT_MAX_ARGS macro       613 src/sentinel.c   #define SENTINEL_SCRIPT_MAX_ARGS 16

- [ ] sentinelScheduleScriptExecution function    614 src/sentinel.c   void sentinelScheduleScriptExecution(char *path, ...) {

- [ ] sentinelGetScriptListNodeByPid function    662 src/sentinel.c   listNode *sentinelGetScriptListNodeByPid(pid_t pid) {

- [ ] sentinelRunPendingScripts function    678 src/sentinel.c   void sentinelRunPendingScripts(void) {

- [ ] sentinelScriptRetryDelay function    731 src/sentinel.c   mstime_t sentinelScriptRetryDelay(int retry_num) {

- [ ] sentinelCollectTerminatedScripts function    742 src/sentinel.c   void sentinelCollectTerminatedScripts(void) {

- [ ] sentinelKillTimedoutScripts function    789 src/sentinel.c   void sentinelKillTimedoutScripts(void) {

- [ ] sentinelPendingScriptsCommand function    809 src/sentinel.c   void sentinelPendingScriptsCommand(redisClient *c) {

- [ ] sentinelCallClientReconfScript function    861 src/sentinel.c   void sentinelCallClientReconfScript(sentinelRedisInstance *master, int role, char *state, sentinelAddr *from, sentinelAddr *to) {

- [ ] createSentinelRedisInstance function    895 src/sentinel.c   sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {

- [ ] releaseSentinelRedisInstance function   1001 src/sentinel.c   void releaseSentinelRedisInstance(sentinelRedisInstance *ri) {

- [ ] sentinelRedisInstanceLookupSlave function   1028 src/sentinel.c   sentinelRedisInstance *sentinelRedisInstanceLookupSlave(

- [ ] sentinelRedisInstanceTypeStr function   1044 src/sentinel.c   const char *sentinelRedisInstanceTypeStr(sentinelRedisInstance *ri) {

- [ ] removeMatchingSentinelsFromMaster function   1068 src/sentinel.c   int removeMatchingSentinelsFromMaster(sentinelRedisInstance *master, char *ip, int port, char *runid) {

- [ ] getSentinelRedisInstanceByAddrAndRunID function   1094 src/sentinel.c   sentinelRedisInstance *getSentinelRedisInstanceByAddrAndRunID(dict *instances, char *ip, int port, char *runid) {

- [ ] sentinelGetMasterByName function   1118 src/sentinel.c   sentinelRedisInstance *sentinelGetMasterByName(char *name) {

- [ ] sentinelAddFlagsToDictOfRedisInstances function   1128 src/sentinel.c   void sentinelAddFlagsToDictOfRedisInstances(dict *instances, int flags) {

- [ ] sentinelDelFlagsToDictOfRedisInstances function   1142 src/sentinel.c   void sentinelDelFlagsToDictOfRedisInstances(dict *instances, int flags) {

- [ ] SENTINEL_RESET_NO_SENTINELS macro      1163 src/sentinel.c   #define SENTINEL_RESET_NO_SENTINELS (1<<0)

- [ ] sentinelResetMaster function   1164 src/sentinel.c   void sentinelResetMaster(sentinelRedisInstance *ri, int flags) {

- [ ] sentinelResetMastersByPattern function   1198 src/sentinel.c   int sentinelResetMastersByPattern(char *pattern, int flags) {

- [ ] sentinelResetMasterAndChangeAddress function   1225 src/sentinel.c   int sentinelResetMasterAndChangeAddress(sentinelRedisInstance *master, char *ip, int port) {

- [ ] sentinelRedisInstanceNoDownFor function   1287 src/sentinel.c   int sentinelRedisInstanceNoDownFor(sentinelRedisInstance *ri, mstime_t ms) {

- [ ] sentinelGetCurrentMasterAddress function   1298 src/sentinel.c   sentinelAddr *sentinelGetCurrentMasterAddress(sentinelRedisInstance *master) {

- [ ] sentinelPropagateDownAfterPeriod function   1316 src/sentinel.c   void sentinelPropagateDownAfterPeriod(sentinelRedisInstance *master) {

- [ ] sentinelHandleConfiguration function   1333 src/sentinel.c   char *sentinelHandleConfiguration(char **argv, int argc) {

- [ ] rewriteConfigSentinelOption function   1452 src/sentinel.c   void rewriteConfigSentinelOption(struct rewriteConfigState *state) {

- [ ] sentinelFlushConfig function   1596 src/sentinel.c   void sentinelFlushConfig(void) {

- [ ] sentinelKillLink function   1619 src/sentinel.c   void sentinelKillLink(sentinelRedisInstance *ri, redisAsyncContext *c) {

- [ ] sentinelDisconnectInstanceFromContext function   1636 src/sentinel.c   void sentinelDisconnectInstanceFromContext(const redisAsyncContext *c) {

- [ ] sentinelLinkEstablishedCallback function   1652 src/sentinel.c   void sentinelLinkEstablishedCallback(const redisAsyncContext *c, int status) {

- [ ] sentinelDisconnectCallback function   1664 src/sentinel.c   void sentinelDisconnectCallback(const redisAsyncContext *c, int status) {

- [ ] sentinelSendAuthIfNeeded function   1675 src/sentinel.c   void sentinelSendAuthIfNeeded(sentinelRedisInstance *ri, redisAsyncContext *c) {

- [ ] sentinelSetClientName function   1691 src/sentinel.c   void sentinelSetClientName(sentinelRedisInstance *ri, redisAsyncContext *c, char *type) {

- [ ] sentinelReconnectInstance function   1705 src/sentinel.c   void sentinelReconnectInstance(sentinelRedisInstance *ri) {

- [ ] sentinelMasterLooksSane function   1774 src/sentinel.c   int sentinelMasterLooksSane(sentinelRedisInstance *master) {

- [ ] sentinelRefreshInstanceInfo function   1783 src/sentinel.c   void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {

- [ ] sentinelInfoReplyCallback function   2027 src/sentinel.c   void sentinelInfoReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {

- [ ] sentinelDiscardReplyCallback function   2043 src/sentinel.c   void sentinelDiscardReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {

- [ ] sentinelPingReplyCallback function   2051 src/sentinel.c   void sentinelPingReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {

- [ ] sentinelPublishReplyCallback function   2090 src/sentinel.c   void sentinelPublishReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {

- [ ] sentinelProcessHelloMessage function   2110 src/sentinel.c   void sentinelProcessHelloMessage(char *hello, int hello_len) {

- [ ] sentinelReceiveHelloMessages function   2200 src/sentinel.c   void sentinelReceiveHelloMessages(redisAsyncContext *c, void *reply, void *privdata) {

- [ ] sentinelSendHello function   2239 src/sentinel.c   int sentinelSendHello(sentinelRedisInstance *ri) {

- [ ] sentinelForceHelloUpdateDictOfRedisInstances function   2281 src/sentinel.c   void sentinelForceHelloUpdateDictOfRedisInstances(dict *instances) {

- [ ] sentinelForceHelloUpdateForMaster function   2302 src/sentinel.c   int sentinelForceHelloUpdateForMaster(sentinelRedisInstance *master) {

- [ ] sentinelSendPing function   2316 src/sentinel.c   int sentinelSendPing(sentinelRedisInstance *ri) {

- [ ] sentinelSendPeriodicCommands function   2333 src/sentinel.c   void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {

- [ ] sentinelFailoverStateStr function   2386 src/sentinel.c   const char *sentinelFailoverStateStr(int state) {

- [ ] addReplySentinelRedisInstance function   2400 src/sentinel.c   void addReplySentinelRedisInstance(redisClient *c, sentinelRedisInstance *ri) {

- [ ] addReplyDictOfRedisInstances function   2587 src/sentinel.c   void addReplyDictOfRedisInstances(redisClient *c, dict *instances) {

- [ ] sentinelGetMasterByNameOrReplyError function   2604 src/sentinel.c   sentinelRedisInstance *sentinelGetMasterByNameOrReplyError(redisClient *c,

- [ ] sentinelCommand  function   2617 src/sentinel.c   void sentinelCommand(redisClient *c) {

- [ ] sentinelInfoCommand function   2792 src/sentinel.c   void sentinelInfoCommand(redisClient *c) {

- [ ] sentinelRoleCommand function   2853 src/sentinel.c   void sentinelRoleCommand(redisClient *c) {

- [ ] sentinelSetCommand function   2871 src/sentinel.c   void sentinelSetCommand(redisClient *c) {

- [ ] sentinelPublishCommand function   2964 src/sentinel.c   void sentinelPublishCommand(redisClient *c) {

- [ ] sentinelCheckSubjectivelyDown function   2976 src/sentinel.c   void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri) {

- [ ] sentinelCheckObjectivelyDown function   3044 src/sentinel.c   void sentinelCheckObjectivelyDown(sentinelRedisInstance *master) {

- [ ] sentinelReceiveIsMasterDownReply function   3081 src/sentinel.c   void sentinelReceiveIsMasterDownReply(redisAsyncContext *c, void *reply, void *privdata) {

- [ ] SENTINEL_ASK_FORCED macro      3123 src/sentinel.c   #define SENTINEL_ASK_FORCED (1<<0)

- [ ] sentinelAskMasterStateToOtherSentinels function   3124 src/sentinel.c   void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {

- [ ] sentinelVoteLeader function   3174 src/sentinel.c   char *sentinelVoteLeader(sentinelRedisInstance *master, uint64_t req_epoch, char *req_runid, uint64_t *leader_epoch) {

- [ ] sentinelLeader   struct     3201 src/sentinel.c   struct sentinelLeader {

- [ ] runid            member     3202 src/sentinel.c   char *runid;

- [ ] votes            member     3203 src/sentinel.c   unsigned long votes;

- [ ] sentinelLeaderIncr function   3208 src/sentinel.c   int sentinelLeaderIncr(dict *counters, char *runid) {

- [ ] sentinelGetLeader function   3230 src/sentinel.c   char *sentinelGetLeader(sentinelRedisInstance *master, uint64_t epoch) {

- [ ] sentinelSendSlaveOf function   3305 src/sentinel.c   int sentinelSendSlaveOf(sentinelRedisInstance *ri, char *host, int port) {

- [ ] sentinelStartFailover function   3362 src/sentinel.c   void sentinelStartFailover(sentinelRedisInstance *master) {

- [ ] sentinelStartFailoverIfNeeded function   3386 src/sentinel.c   int sentinelStartFailoverIfNeeded(sentinelRedisInstance *master) {

- [ ] compareSlavesForPromotion function   3448 src/sentinel.c   int compareSlavesForPromotion(const void *a, const void *b) {

- [ ] sentinelSelectSlave function   3476 src/sentinel.c   sentinelRedisInstance *sentinelSelectSlave(sentinelRedisInstance *master) {

- [ ] sentinelFailoverWaitStart function   3520 src/sentinel.c   void sentinelFailoverWaitStart(sentinelRedisInstance *ri) {

- [ ] sentinelFailoverSelectSlave function   3551 src/sentinel.c   void sentinelFailoverSelectSlave(sentinelRedisInstance *ri) {

- [ ] sentinelFailoverSendSlaveOfNoOne function   3570 src/sentinel.c   void sentinelFailoverSendSlaveOfNoOne(sentinelRedisInstance *ri) {

- [ ] sentinelFailoverWaitPromotion function   3598 src/sentinel.c   void sentinelFailoverWaitPromotion(sentinelRedisInstance *ri) {

- [ ] sentinelFailoverDetectEnd function   3607 src/sentinel.c   void sentinelFailoverDetectEnd(sentinelRedisInstance *master) {

- [ ] sentinelFailoverReconfNextSlave function   3672 src/sentinel.c   void sentinelFailoverReconfNextSlave(sentinelRedisInstance *master) {

- [ ] sentinelFailoverSwitchToPromotedSlave function   3734 src/sentinel.c   void sentinelFailoverSwitchToPromotedSlave(sentinelRedisInstance *master) {

- [ ] sentinelFailoverStateMachine function   3745 src/sentinel.c   void sentinelFailoverStateMachine(sentinelRedisInstance *ri) {

- [ ] sentinelAbortFailover function   3774 src/sentinel.c   void sentinelAbortFailover(sentinelRedisInstance *ri) {

- [ ] sentinelHandleRedisInstance function   3793 src/sentinel.c   void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {

- [ ] sentinelHandleDictOfRedisInstances function   3829 src/sentinel.c   void sentinelHandleDictOfRedisInstances(dict *instances) {

- [ ] sentinelCheckTiltCondition function   3872 src/sentinel.c   void sentinelCheckTiltCondition(void) {

- [ ] sentinelTimer    function   3884 src/sentinel.c   void sentinelTimer(void) {


<template>
    <div class="tournament-table">
        <div class="mobile-warning" v-if="isMobile">
            <p>
                申し訳ありませんが、このアプリはスマートフォンに対応していません。
            </p>
            <p>パソコンでご利用ください。</p>
        </div>
        <div v-else>
            <h2 class="title">🏆 総当たり表 🏆</h2>

            <!-- 設定ボタン -->
            <div class="team-management-button-container">
                <button
                    @click="showModal = true"
                    class="team-management-button"
                >
                    設定
                </button>
            </div>

            <div class="view-toggle">
                <button
                    @click="viewMode = 'all'"
                    :class="{ active: viewMode === 'all' }"
                >
                    全体表示
                </button>
                <button
                    @click="viewMode = 'individual'"
                    :class="{ active: viewMode === 'individual' }"
                >
                    個別表示
                </button>
            </div>

            <div v-if="viewMode === 'individual'">
                <div class="tabs">
                    <button
                        v-for="(team, index) in teams"
                        :key="team"
                        @click="selectedTeam = index"
                        :class="{ active: selectedTeam === index }"
                    >
                        {{ team }}
                    </button>
                </div>
                <table v-if="selectedTeam !== null" class="individual-table">
                    <thead>
                        <tr>
                            <th>相手</th>
                            <th>結果</th>
                            <th>{{ teams[selectedTeam] }}のスコア</th>
                            <th>相手のスコア</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr
                            v-for="(result, index) in results[selectedTeam]"
                            :key="index"
                        >
                            <template v-if="index !== selectedTeam">
                                <td>{{ teams[index] }}</td>
                                <td :class="getResultClass(result.result)">
                                    {{ getResultText(result.result) }}
                                </td>
                                <td>
                                    <input
                                        v-model="result.myScore"
                                        type="number"
                                        min="0"
                                        @input="
                                            updateResult(selectedTeam, index)
                                        "
                                    />
                                </td>
                                <td>
                                    <input
                                        v-model="result.opponentScore"
                                        type="number"
                                        min="0"
                                        @input="
                                            updateResult(selectedTeam, index)
                                        "
                                    />
                                </td>
                            </template>
                        </tr>
                    </tbody>
                    <tfoot>
                        <tr>
                            <td colspan="2">合計</td>
                            <td>
                                合計点数: {{ getPoints(selectedTeam) }}, 勝:
                                {{ getWins(selectedTeam) }}, 負:
                                {{ getLosses(selectedTeam) }}
                            </td>
                        </tr>
                    </tfoot>
                </table>
            </div>

            <div v-if="viewMode === 'all'">
                <table class="all-table">
                    <thead>
                        <tr>
                            <th>私 \ 相手</th>
                            <th v-for="team in teams" :key="team">
                                {{ team }}
                            </th>
                            <th>合計点数</th>
                            <th>勝</th>
                            <th>負</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(row, index) in results" :key="index">
                            <td>{{ teams[index] }}</td>
                            <td
                                v-for="(result, colIndex) in row"
                                :key="colIndex"
                                :class="getResultClass(result.result)"
                            >
                                <template v-if="index !== colIndex">
                                    {{ getResultText(result.result) }}
                                    <template
                                        v-if="
                                            result.myScore ||
                                            result.opponentScore
                                        "
                                    >
                                        {{ result.myScore }}：{{
                                            result.opponentScore
                                        }}
                                    </template>
                                </template>
                                <template v-else> - </template>
                            </td>
                            <td
                                :class="{
                                    'highest-score':
                                        isHighestScore(index) &&
                                        getPoints(index) > 0,
                                }"
                            >
                                {{ getPoints(index) }}
                            </td>
                            <td
                                :class="{
                                    'most-wins':
                                        isMostWins(index) && getWins(index) > 0,
                                }"
                            >
                                {{ getWins(index) }}
                            </td>
                            <td>{{ getLosses(index) }}</td>
                        </tr>
                    </tbody>
                </table>
            </div>

            <!-- モーダルウィンドウ -->
            <div v-if="showModal" class="modal-overlay">
                <div class="modal-content">
                    <h3 class="modal-title">設定</h3>

                    <!-- プレイヤー名追加フォーム -->
                    <div class="add-team-form">
                        <textarea
                            class="add-team-textarea"
                            v-model="newTeamNames"
                            placeholder="新しいプレイヤー名名を1行に1つずつ入力してください"
                            rows="5"
                        ></textarea>
                        <button @click="addTeams" class="add-team-button">
                            プレイヤー名を追加
                        </button>
                    </div>

                    <!-- プレイヤー名リスト -->
                    <div class="team-list-container">
                        <h4>現在のプレイヤー名</h4>
                        <div class="team-list">
                            <div
                                v-for="(team, index) in teams"
                                :key="index"
                                class="team-item"
                            >
                                {{ team }}
                                <button
                                    @click="removeTeam(index)"
                                    class="remove-team"
                                >
                                    ×
                                </button>
                            </div>
                        </div>
                    </div>

                    <!-- リセットボタン -->
                    <div class="reset-button-container">
                        <button @click="resetScores" class="reset-button">
                            得点をリセット
                        </button>
                    </div>

                    <!-- モーダルを閉じるボタン -->
                    <button
                        @click="showModal = false"
                        class="close-modal-button"
                    >
                        閉じる
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    data() {
        return {
            teams: [],
            results: [],
            selectedTeam: null,
            viewMode: "individual",
            newTeamNames: "",
            showModal: false,
            isMobile: false,
        };
    },
    created() {
        this.loadData();
        if (this.teams.length === 0) {
            // デフォルトのプレイヤー名を追加するか、ユーザーにプレイヤー名追加を促す
            this.addTeam("プレイヤー1");
            this.addTeam("プレイヤー2");
        }
        this.checkMobile();
        window.addEventListener("resize", this.checkMobile);
    },
    beforeUnmount() {
        window.removeEventListener("resize", this.checkMobile);
    },
    methods: {
        checkMobile() {
            this.isMobile = window.innerWidth <= 768;
        },
        loadData() {
            const savedData = localStorage.getItem("tournamentData");
            if (savedData) {
                const parsedData = JSON.parse(savedData);
                this.teams = parsedData.teams;
                this.results = parsedData.results;
            } else {
                this.initializeResults();
            }
            this.selectedTeam = 0;
        },
        saveData() {
            const dataToSave = {
                teams: this.teams,
                results: this.results,
            };
            localStorage.setItem("tournamentData", JSON.stringify(dataToSave));
        },
        addTeams() {
            const newTeams = this.newTeamNames
                .split("\n")
                .filter((name) => name.trim() !== "");
            if (newTeams.length > 0) {
                newTeams.forEach((teamName) => {
                    this.addTeam(teamName.trim());
                });
                this.newTeamNames = ""; // テキストエリアをクリア
            }
        },
        addTeam(teamName) {
            if (teamName) {
                const newTeamIndex = this.teams.length;
                this.teams.push(teamName);

                // 新しい行を追加
                this.results.push(
                    new Array(newTeamIndex + 1).fill().map(() => ({
                        result: "",
                        myScore: "",
                        opponentScore: "",
                    }))
                );

                // 既存の行に新しい列を追加
                this.results.forEach((row, index) => {
                    if (index < newTeamIndex) {
                        row.push({
                            result: "",
                            myScore: "",
                            opponentScore: "",
                        });
                    }
                });

                this.saveData();
            }
        },
        removeTeam(index) {
            if (confirm(`本当に「${this.teams[index]}」を削除しますか？`)) {
                this.teams.splice(index, 1);

                // インデックスの行を削除
                this.results.splice(index, 1);

                // 各行からインデックスの列を削除
                this.results.forEach((row) => {
                    row.splice(index, 1);
                });

                if (this.selectedTeam >= this.teams.length) {
                    this.selectedTeam = this.teams.length - 1;
                }
                this.saveData();
            }
        },
        initializeResults() {
            this.results = this.teams.map(() =>
                this.teams.map(() => ({
                    result: "",
                    myScore: "",
                    opponentScore: "",
                }))
            );
            this.saveData();
        },
        updateResult(index, colIndex) {
            const result = this.results[index][colIndex];
            const myScore = parseInt(result.myScore) || 0;
            const opponentScore = parseInt(result.opponentScore) || 0;

            if (myScore > opponentScore) {
                result.result = "win";
            } else if (myScore < opponentScore) {
                result.result = "lose";
            } else if (
                myScore === opponentScore &&
                (myScore !== 0 || opponentScore !== 0)
            ) {
                result.result = "draw";
            } else {
                result.result = "";
            }

            // 反対側のセルも更新
            this.results[colIndex][index] = {
                result:
                    result.result === "win"
                        ? "lose"
                        : result.result === "lose"
                        ? "win"
                        : result.result,
                myScore: result.opponentScore,
                opponentScore: result.myScore,
            };
            this.saveData();
        },
        resetScores() {
            if (
                confirm(
                    "本当に全ての得点をリセットしますか？この操作は元に戻せません。"
                )
            ) {
                this.results = this.teams.map(() =>
                    this.teams.map(() => ({
                        result: "",
                        myScore: "",
                        opponentScore: "",
                    }))
                );
                this.saveData();
                this.showModal = false; // モーダルを閉じる
            }
        },
        getPoints(teamIndex) {
            return this.results[teamIndex].reduce((points, result) => {
                return points + (parseInt(result.myScore) || 0);
            }, 0);
        },
        getWins(teamIndex) {
            return this.results[teamIndex].filter(
                (result) => result.result === "win"
            ).length;
        },
        getLosses(teamIndex) {
            return this.results[teamIndex].filter(
                (result) => result.result === "lose"
            ).length;
        },
        getResultClass(result) {
            if (result === "win") return "win";
            if (result === "lose") return "lose";
            if (result === "draw") return "draw";
            return "";
        },
        getResultText(result) {
            if (result === "win") return "勝ち";
            if (result === "lose") return "負け";
            if (result === "draw") return "引き分け";
            return "-";
        },
        isHighestScore(index) {
            const scores = this.teams.map((_, i) => this.getPoints(i));
            const maxScore = Math.max(...scores);
            return this.getPoints(index) === maxScore && maxScore > 0;
        },
        isMostWins(index) {
            const wins = this.teams.map((_, i) => this.getWins(i));
            const maxWins = Math.max(...wins);
            return this.getWins(index) === maxWins && maxWins > 0;
        },
    },
};
</script>

<style scoped>
.tournament-table {
    font-family: "Arial Rounded MT Bold", "Helvetica Rounded", Arial, sans-serif;
    background-color: #fff5f5;
    padding: 20px;
    border-radius: 15px;
    /* box-shadow: 0 0 15px rgba(0, 0, 0, 0.1); */
}

.title {
    color: #ff6b6b;
    text-align: center;
    font-size: 2em;
    margin-bottom: 20px;
}

table {
    border-collapse: separate;
    border-spacing: 0;
    border-radius: 10px;
    overflow: hidden;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
    margin-top: 20px;
}

th,
td {
    border: 1px solid #ffd3d3;
    padding: 12px;
    text-align: center;
}

th {
    background-color: #ffb3ba;
    color: white;
}

select,
input {
    width: 100%;
    max-width: 100px;
    margin: 2px;
    padding: 5px;
    border: 2px solid #ffb3ba;
    border-radius: 5px;
    font-size: 0.9em;
    box-sizing: border-box;
}

input[type="number"] {
    max-width: 70px;
}

.tabs,
.view-toggle {
    display: flex;
    justify-content: center;
    margin-bottom: 20px;
}

.tabs button,
.view-toggle button {
    padding: 10px 20px;
    margin: 0 5px;
    border: none;
    background-color: #ffc3a0;
    color: white;
    cursor: pointer;
    border-radius: 20px;
    transition: all 0.3s ease;
}

.tabs button.active,
.view-toggle button.active {
    background-color: #ff6b6b;
    transform: scale(1.05);
}

.individual-table,
.all-table {
    width: 100%;
    max-width: 90%;
    margin: 20px auto;
    table-layout: fixed;
}

.individual-table th,
.individual-table td,
.all-table th,
.all-table td {
    padding: 10px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.individual-table th:first-child,
.individual-table td:first-child,
.all-table th:first-child,
.all-table td:first-child {
    width: 200px;
}

.individual-table th:nth-child(2),
.individual-table td:nth-child(2) {
    width: 80px;
}

.individual-table th:nth-child(3),
.individual-table th:nth-child(4),
.individual-table td:nth-child(3),
.individual-table td:nth-child(4) {
    width: 100px;
}

.all-table th,
.all-table td {
    width: calc((100% - 120px - 240px) / 7);
}

.all-table th:last-child,
.all-table th:nth-last-child(2),
.all-table th:nth-last-child(3),
.all-table td:last-child,
.all-table td:nth-last-child(2),
.all-table td:nth-last-child(3) {
    width: 80px;
}

.win {
    background-color: #baffc9;
}

.lose {
    background-color: #ffb3ba;
}

.draw {
    background-color: #ffffba;
}

/* アニメーション効果 */
@keyframes fadeIn {
    from {
        opacity: 0;
    }
    to {
        opacity: 1;
    }
}

.tournament-table {
    animation: fadeIn 0.5s ease-in-out;
}

.highest-score {
    background-color: #ffd700; /* 金色 */
    font-weight: bold;
}

.most-wins {
    background-color: #c0c0c0; /* 銀色 */
    font-weight: bold;
}

.reset-button-container {
    display: flex;
    justify-content: space-between;
    margin-top: 20px;
}

.reset-button {
    padding: 10px 20px;
    background-color: #ff4136;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

.reset-button:hover {
    background-color: #ff1a1a;
}

.team-management {
    margin-bottom: 20px;
}

.team-list {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    margin-top: 10px;
}

.team-item {
    background-color: #ffb3ba;
    color: white;
    padding: 5px 10px;
    margin: 5px;
    border-radius: 15px;
    display: flex;
    align-items: center;
}

.remove-team {
    background-color: transparent;
    border: none;
    color: white;
    font-size: 1.2em;
    cursor: pointer;
    margin-left: 5px;
}

.remove-team:hover {
    color: #ff4136;
}

.team-management-button-container {
    display: flex;
    justify-content: center;
    margin-bottom: 20px;
}

.team-management-button {
    padding: 10px 20px;
    background-color: #4caf50;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

.team-management-button:hover {
    background-color: #45a049;
}

.modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
}

.modal-content {
    background-color: white;
    padding: 30px;
    border-radius: 15px;
    width: 90%;
    max-width: 500px;
    max-height: 80vh;
    overflow-y: auto;
}

.modal-title {
    color: #ff6b6b;
    text-align: center;
    margin-bottom: 20px;
}

.add-team-form {
    margin-bottom: 20px;
    width: 100%;
}

.add-team-textarea {
    width: 100%;
    height: 100px;
    padding: 10px;
    margin-bottom: 10px;
    border: 2px solid #ffb3ba;
    border-radius: 5px;
    resize: vertical;
    box-sizing: border-box;
}

.add-team-button {
    width: 100%;
    padding: 10px;
    background-color: #4caf50;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.add-team-button:hover {
    background-color: #45a049;
}

.team-list-container {
    margin-bottom: 20px;
}

.team-list-container h4 {
    margin-bottom: 10px;
    color: #ff6b6b;
}

.team-list {
    max-height: 200px;
    overflow-y: auto;
    border: 1px solid #ffb3ba;
    border-radius: 5px;
    padding: 10px;
}

.team-item {
    background-color: #ffb3ba;
    color: white;
    padding: 5px 10px;
    margin: 5px 0;
    border-radius: 15px;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.remove-team {
    background-color: transparent;
    border: none;
    color: white;
    font-size: 1.2em;
    cursor: pointer;
    transition: color 0.3s;
}

.remove-team:hover {
    color: #ff4136;
}

.reset-button-container {
    margin-bottom: 20px;
}

.reset-button {
    width: 100%;
    padding: 10px;
    background-color: #ff4136;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.reset-button:hover {
    background-color: #ff1a1a;
}

.close-modal-button {
    width: 100%;
    padding: 10px;
    background-color: #ccc;
    color: #333;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.close-modal-button:hover {
    background-color: #bbb;
}
.add-team-textarea {
    width: 100%;
    height: 100px;
    padding: 10px;
    margin-bottom: 10px;
    border: 2px solid #ffb3ba;
    border-radius: 5px;
    resize: vertical;
    box-sizing: border-box;
    min-height: 100px;
}
</style>

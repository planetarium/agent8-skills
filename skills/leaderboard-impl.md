---
name: leaderboard-impl
description: A method to implement leaderboards and rankings in a game. Explain how to store on a server and display it on a client (using `gameserver-sdk` skill).
---

# Leaderboard Implementation

This document explains how to implement leaderboards using Agent8 GameServer SDK's Collection functionality. We cover two main approaches.

## Prerequisites

**Important:** This implementation uses the Agent8 GameServer SDK. Before implementing a leaderboard, you must:

1. Read and understand the `gameserver-sdk` skill
2. Set up Agent8 GameServer in your project
3. Create a `server.js` file in your project root

The leaderboard uses global Collections (`$global.addCollectionItem`, `$global.getCollectionItems`, etc.) and requires understanding of the SDK's state management system.

## 1. Multiple Scores per User Approach

This approach stores all game play records for each user, allowing users to submit multiple scores.

### Server Code Implementation

```js filename='server.js'
class Server {
  /**
   * Submits a new score to the leaderboard.
   * @param {number} score The player's score.
   * @param {string} nickname The player's nickname.
   * @returns {Promise<Item>} The newly created leaderboard entry.
   */
  async submitScore(score, nickname) {
    if (typeof score !== "number" || score < 0) {
      throw new Error("Invalid score.");
    }
    if (
      typeof nickname !== "string" ||
      nickname.length < 1 ||
      nickname.length > 15
    ) {
      throw new Error("Nickname must be between 1 and 15 characters.");
    }

    const entry = {
      account: $sender.account,
      score,
      nickname,
      createdAt: Date.now(),
    };

    return await $global.addCollectionItem("rankings", entry);
  }

  /**
   * Retrieves the top 20 rankings.
   * @returns {Promise<Item[]>} A list of the top 20 players.
   */
  async getTopRankings() {
    return await $global.getCollectionItems("rankings", {
      orderBy: [{ field: "score", direction: "desc" }],
      limit: 20,
    });
  }

  /**
   * Retrieves the current player's best score and rank.
   * @returns {Promise<{bestEntry: Item | null, rank: number}>} The player's best score entry and their rank.
   */
  async getMyBestRank() {
    const myRankings = await this.getMyAllRankings();

    if (myRankings.length === 0) {
      return { bestEntry: null, rank: -1 };
    }

    // getMyAllRankings returns rankings sorted by score desc, so first item is the best
    const bestEntry = myRankings[0];

    const higherRankCount = await $global.countCollectionItems("rankings", {
      filters: [{ field: "score", operator: ">", value: bestEntry.score }],
    });

    return {
      bestEntry,
      rank: higherRankCount + 1,
    };
  }

  /**
   * Retrieves all rankings for the current player.
   * @returns {Promise<Item[]>} All rankings for the current player.
   */
  async getMyAllRankings() {
    const myRankings = await $global.getCollectionItems("rankings", {
      filters: [{ field: "account", operator: "==", value: $sender.account }],
    });

    // Sort on client side since we can't use both filters and orderBy
    return myRankings.sort((a, b) => b.score - a.score);
  }
}
```

### Client-Side Implementation

#### Leaderboard Dialog Component

Create a separate file for the leaderboard component:

```tsx filename='components/LeaderboardDialog.tsx'
import React, { useState, useEffect } from "react";
import { useGameServer } from "@agent8/gameserver";

interface LeaderboardDialogProps {
  isOpen: boolean;
  onClose: () => void;
}

interface ScoreEntry {
  __id: string;
  account: string;
  nickname: string;
  score: number;
}

const LeaderboardDialog: React.FC<LeaderboardDialogProps> = ({
  isOpen,
  onClose,
}) => {
  const { connected, server } = useGameServer();
  const [topRanks, setTopRanks] = useState<ScoreEntry[]>([]);
  const [myRank, setMyRank] = useState<{
    bestEntry: ScoreEntry | null;
    rank: number;
  } | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!isOpen || !connected) return;

    const fetchLeaderboard = async () => {
      try {
        setLoading(true);
        const [top, my] = await Promise.all([
          server.remoteFunction("getTopRankings"),
          server.remoteFunction("getMyBestRank"),
        ]);
        setTopRanks(top as ScoreEntry[]);
        setMyRank(my as { bestEntry: ScoreEntry | null; rank: number });
      } catch (error) {
        console.error("Failed to fetch leaderboard:", error);
      } finally {
        setLoading(false);
      }
    };

    fetchLeaderboard();
  }, [isOpen, connected, server]);

  if (!isOpen) return null;

  return (
    <div
      className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"
      onClick={onClose}
    >
      <div
        className="bg-white rounded-lg p-6 max-w-md w-full mx-4"
        onClick={(e) => e.stopPropagation()}
      >
        <div className="flex justify-between items-center mb-4">
          <h1 className="text-xl font-bold">Leaderboard</h1>
          <button
            onClick={onClose}
            className="text-gray-500 hover:text-gray-700"
          >
            ✕
          </button>
        </div>

        {loading ? (
          <div className="text-center py-8">Loading...</div>
        ) : (
          <div>
            <div className="space-y-2 mb-6">
              {topRanks.map((entry, index) => (
                <div
                  key={entry.__id}
                  className={`flex justify-between items-center p-2 rounded ${
                    entry.account === server.account
                      ? "bg-blue-100"
                      : "bg-gray-50"
                  }`}
                >
                  <div className="flex items-center gap-2">
                    <span className="font-medium">#{index + 1}</span>
                    <span>{entry.nickname}</span>
                  </div>
                  <span className="font-bold">
                    {entry.score.toLocaleString()}
                  </span>
                </div>
              ))}
            </div>

            {myRank?.bestEntry && myRank.rank > 20 && (
              <div className="border-t pt-4">
                <h2 className="font-medium mb-2">Your Best</h2>
                <div className="flex justify-between items-center p-2 bg-blue-50 rounded">
                  <div className="flex items-center gap-2">
                    <span className="font-medium">#{myRank.rank}</span>
                    <span>{myRank.bestEntry.nickname}</span>
                  </div>
                  <span className="font-bold">
                    {myRank.bestEntry.score.toLocaleString()}
                  </span>
                </div>
              </div>
            )}
          </div>
        )}

        <button
          onClick={onClose}
          className="w-full mt-4 px-4 py-2 bg-gray-600 text-white rounded hover:bg-gray-700"
        >
          Close
        </button>
      </div>
    </div>
  );
};

export default LeaderboardDialog;
```

#### Score Submission Component

```tsx filename='components/GameOverDialog.tsx'
import React, { useState } from "react";
import { useGameServer } from "@agent8/gameserver";

interface GameOverDialogProps {
  score: number;
  onRestart: () => void;
  onScoreSubmitted: () => void;
}

const GameOverDialog: React.FC<GameOverDialogProps> = ({
  score,
  onRestart,
  onScoreSubmitted,
}) => {
  const { connected, server } = useGameServer();
  const [nickname, setNickname] = useState("");
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState("");

  const handleSubmit = async () => {
    if (!connected) {
      setError("Not connected to server.");
      return;
    }
    if (nickname.trim().length === 0) {
      setError("Nickname cannot be empty.");
      return;
    }
    if (nickname.length > 15) {
      setError("Nickname cannot exceed 15 characters.");
      return;
    }

    setIsSubmitting(true);
    setError("");
    try {
      await server.remoteFunction("submitScore", [score, nickname.trim()]);
      onScoreSubmitted(); // recommend: close gameOver dialog and show leaderboard dialog
    } catch (e: any) {
      setError(e.message || "Failed to submit score.");
      setIsSubmitting(false);
    }
  };

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg p-6 max-w-sm w-full mx-4">
        <h1 className="text-2xl font-bold text-center mb-4">Game Over</h1>
        <p className="text-center text-lg mb-6">
          Final Score: {score.toLocaleString("en-US")}
        </p>

        <div className="space-y-4">
          <input
            type="text"
            value={nickname}
            onChange={(e) => setNickname(e.target.value)}
            placeholder="Enter your nickname"
            className="w-full px-3 py-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
            maxLength={15}
            disabled={isSubmitting}
          />

          <button
            onClick={handleSubmit}
            disabled={!connected || isSubmitting}
            className="w-full px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
          >
            {isSubmitting ? "Submitting..." : "Submit Score"}
          </button>

          {error && <p className="text-red-500 text-sm text-center">{error}</p>}

          <button
            onClick={onRestart}
            className="w-full px-4 py-2 bg-gray-600 text-white rounded hover:bg-gray-700"
          >
            Play Again
          </button>
        </div>
      </div>
    </div>
  );
};

export default GameOverDialog;
```

#### Usage Example

```tsx filename='YourGameComponent.tsx'
import React, { useState } from "react";
import LeaderboardDialog from "./LeaderboardDialog";
import GameOverDialog from "./GameOverDialog";

const YourGame = () => {
  const [showLeaderboard, setShowLeaderboard] = useState(false);
  const [showGameOver, setShowGameOver] = useState(false);
  const [finalScore, setFinalScore] = useState(0);

  const handleGameOver = (score: number) => {
    setFinalScore(score);
    setShowGameOver(true);
  };

  const handleScoreSubmitted = () => {
    setShowGameOver(false);
    setShowLeaderboard(true);
  };

  return (
    <div>
      {/* Your game content */}
      <button
        onClick={() => setShowLeaderboard(true)}
        className="px-4 py-2 bg-blue-600 text-white rounded"
      >
        Show Leaderboard
      </button>

      <LeaderboardDialog
        isOpen={showLeaderboard}
        onClose={() => setShowLeaderboard(false)}
      />

      <GameOverDialog
        score={finalScore}
        onRestart={() => setShowGameOver(false)}
        onScoreSubmitted={handleScoreSubmitted}
      />
    </div>
  );
};
```

## 2. Best Score Only Approach

This approach maintains only one highest score per user. You only need to modify the `submitScore` function from the basic code above.

### Key Difference: submitScore Function

Unlike the basic approach, this only updates when the new score is higher than the existing best score, and deletes all existing scores to prevent duplicates.

```js filename='server.js'
  /**
   * Submits a score, updating only if it's higher than the current best.
   * @param {number} score The player's score.
   * @param {string} nickname The player's nickname.
   * @returns {Promise<{updated: boolean, entry: Item | null, message: string}>} Update result.
   */
  async submitScore(score, nickname) {
    if (typeof score !== "number" || score < 0) {
      throw new Error("Invalid score.");
    }
    if (
      typeof nickname !== "string" ||
      nickname.length < 1 ||
      nickname.length > 15
    ) {
      throw new Error("Nickname must be between 1 and 15 characters.");
    }

    // Get all current rankings (sorted desc by score)
    const myRankings = await this.getMyAllRankings();
    const currentBest = myRankings.length > 0 ? myRankings[0] : null;

    // If no previous score or new score is higher
    if (!currentBest || score > currentBest.score) {
      // Delete all existing rankings to ensure only one ranking per user
      for (const rankingEntry of myRankings) {
        await $global.deleteCollectionItem("rankings", rankingEntry.__id);
      }

      // Add new score
      const entry = {
        account: $sender.account,
        score,
        nickname,
        createdAt: Date.now(),
        updatedAt: Date.now(),
      };

      const newEntry = await $global.addCollectionItem("rankings", entry);

      return {
        updated: true,
        entry: newEntry,
        message: currentBest
          ? `New best score! Previous: ${currentBest.score}`
          : `First score recorded!`,
      };
    } else {
      return {
        updated: false,
        entry: currentBest,
        message: `Score ${score} is not higher than your best: ${currentBest.score}`,
      };
    }
  }
```

**All other functions (`getTopRankings`, `getMyBestRank`, `getMyAllRankings`) remain the same as the basic approach above.**

### Key Difference: GameOverDialog Component

For the best score only approach, the GameOverDialog needs to check if the current score is higher than the player's best score before showing the submission form:

```tsx filename='components/GameOverDialog.tsx'
import React, { useState, useEffect } from "react";
import { useGameServer } from "@agent8/gameserver";

interface GameOverDialogProps {
  score: number;
  onRestart: () => void;
  onScoreSubmitted: () => void;
}

const GameOverDialog: React.FC<GameOverDialogProps> = ({
  score,
  onRestart,
  onScoreSubmitted,
}) => {
  const { connected, server } = useGameServer();
  const [nickname, setNickname] = useState("");
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState("");
  const [myBestScore, setMyBestScore] = useState<number | null>(null);
  const [loading, setLoading] = useState(true);

  // Check current best score when dialog opens
  useEffect(() => {
    if (!connected) return;

    const fetchMyBestScore = async () => {
      try {
        setLoading(true);
        const myRank = await server.remoteFunction("getMyBestRank");
        setMyBestScore(myRank.bestEntry?.score || null);
      } catch (error) {
        console.error("Failed to fetch best score:", error);
      } finally {
        setLoading(false);
      }
    };

    fetchMyBestScore();
  }, [connected, server]);

  const handleSubmit = async () => {
    if (!connected) {
      setError("Not connected to server.");
      return;
    }
    if (nickname.trim().length === 0) {
      setError("Nickname cannot be empty.");
      return;
    }
    if (nickname.length > 15) {
      setError("Nickname cannot exceed 15 characters.");
      return;
    }

    setIsSubmitting(true);
    setError("");
    try {
      const result = await server.remoteFunction("submitScore", [
        score,
        nickname.trim(),
      ]);
      if (result.updated) {
        onScoreSubmitted(); // New high score submitted
      }
    } catch (e: any) {
      setError(e.message || "Failed to submit score.");
      setIsSubmitting(false);
    }
  };

  if (loading) {
    return (
      <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
        <div className="bg-white rounded-lg p-6 max-w-sm w-full mx-4">
          <div className="text-center py-8">Loading...</div>
        </div>
      </div>
    );
  }

  const isNewHighScore = myBestScore === null || score > myBestScore;

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg p-6 max-w-sm w-full mx-4">
        <h1 className="text-2xl font-bold text-center mb-4">Game Over</h1>

        <div className="text-center mb-6">
          <p className="text-lg mb-2">
            Final Score:{" "}
            <span className="font-bold text-blue-600">
              {score.toLocaleString("en-US")}
            </span>
          </p>

          {myBestScore !== null && (
            <p className="text-sm text-gray-600">
              Your Best: {myBestScore.toLocaleString("en-US")}
            </p>
          )}
        </div>

        <div className="space-y-4">
          {isNewHighScore ? (
            <>
              {myBestScore !== null && (
                <div className="bg-green-100 border border-green-300 rounded p-3 text-center">
                  <p className="text-green-800 font-medium">
                    🎉 New High Score!
                  </p>
                </div>
              )}

              <input
                type="text"
                value={nickname}
                onChange={(e) => setNickname(e.target.value)}
                placeholder="Enter your nickname"
                className="w-full px-3 py-2 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
                maxLength={15}
                disabled={isSubmitting}
              />

              <button
                onClick={handleSubmit}
                disabled={!connected || isSubmitting || !nickname.trim()}
                className="w-full px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
              >
                {isSubmitting ? "Submitting..." : "Submit High Score"}
              </button>

              {error && (
                <p className="text-red-500 text-sm text-center">{error}</p>
              )}
            </>
          ) : (
            <div className="bg-gray-100 border border-gray-300 rounded p-3 text-center">
              <p className="text-gray-700">
                Score not high enough to save.
                <br />
                Keep trying to beat your best!
              </p>
            </div>
          )}

          <button
            onClick={onRestart}
            className="w-full px-4 py-2 bg-gray-600 text-white rounded hover:bg-gray-700"
          >
            Play Again
          </button>
        </div>
      </div>
    </div>
  );
};

export default GameOverDialog;
```

**Other components (LeaderboardDialog) remain the same as the basic approach above.**

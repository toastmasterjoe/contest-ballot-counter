CREATE TABLE wp_tm_contests (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    date DATETIME NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE wp_tm_contestants (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    contest_id BIGINT UNSIGNED NOT NULL,
    name VARCHAR(255) NOT NULL,
    club VARCHAR(255) NULL,
    speaking_order INT NOT NULL,
    PRIMARY KEY (id),
    KEY contest_id (contest_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE wp_tm_judges (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    wp_user_id BIGINT UNSIGNED NOT NULL,
    contest_id BIGINT UNSIGNED NOT NULL,
    is_tiebreaker TINYINT(1) NOT NULL DEFAULT 0,
    PRIMARY KEY (id),
    KEY contest_id (contest_id),
    KEY wp_user_id (wp_user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE wp_tm_ballots (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    contest_id BIGINT UNSIGNED NOT NULL,
    judge_id BIGINT UNSIGNED NOT NULL,
    rank_1 BIGINT UNSIGNED NOT NULL,
    rank_2 BIGINT UNSIGNED NOT NULL,
    rank_3 BIGINT UNSIGNED NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    PRIMARY KEY (id),
    KEY contest_id (contest_id),
    KEY judge_id (judge_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE wp_tm_tiebreaker_ballots (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    contest_id BIGINT UNSIGNED NOT NULL,
    judge_id BIGINT UNSIGNED NOT NULL,
    ranking_json LONGTEXT NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id),
    KEY contest_id (contest_id),
    KEY judge_id (judge_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

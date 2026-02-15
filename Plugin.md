
---

# ✅ **UNIFIED WORDPRESS PLUGIN (FULL SKELETON + ALL ENDPOINTS + TALLY LOGIC)**

This is the complete, merged version.

---

## **plugin.php**

```php
<?php
/**
 * Plugin Name: TM Contest Ballots
 * Description: REST backend for Toastmasters contest ballots (judges, tiebreakers, tally, purge).
 * Version: 1.0.0
 * Author: Your Name
 */

if (!defined('ABSPATH')) exit;

class TM_Contest_Ballots_Plugin {

    public function __construct() {
        register_activation_hook(__FILE__, [$this, 'activate']);
        add_action('rest_api_init', [$this, 'register_routes']);
    }

    public function activate() {
        require_once __DIR__ . '/includes/class-tm-contest-migrations.php';
        TM_Contest_Migrations::run();
    }

    public function register_routes() {
        require_once __DIR__ . '/includes/class-tm-contest-rest-controller.php';
        (new TM_Contest_REST_Controller())->register_routes();
    }
}

new TM_Contest_Ballots_Plugin();
```

---

# **includes/class-tm-contest-rest-controller.php**  
### ⭐ This version includes ALL endpoints + tally logic.

```php
<?php

class TM_Contest_REST_Controller extends WP_REST_Controller {

    public function __construct() {
        $this->namespace = 'contest/v1';
    }

    public function register_routes() {

        /* -----------------------------
         * Contest list for judge
         * ----------------------------- */
        register_rest_route($this->namespace, '/contests', [
            [
                'methods'  => WP_REST_Server::READABLE,
                'callback' => [$this, 'get_contests'],
                'permission_callback' => [$this, 'permissions_judge'],
            ],
        ]);

        /* -----------------------------
         * Contestants for contest
         * ----------------------------- */
        register_rest_route($this->namespace, '/contests/(?P<id>\d+)/contestants', [
            [
                'methods'  => WP_REST_Server::READABLE,
                'callback' => [$this, 'get_contestants'],
                'permission_callback' => [$this, 'permissions_judge'],
            ],
        ]);

        /* -----------------------------
         * Submit standard ballot
         * ----------------------------- */
        register_rest_route($this->namespace, '/ballot', [
            [
                'methods'  => WP_REST_Server::CREATABLE,
                'callback' => [$this, 'submit_ballot'],
                'permission_callback' => [$this, 'permissions_judge'],
            ],
        ]);

        /* -----------------------------
         * Submit tiebreaker ballot
         * ----------------------------- */
        register_rest_route($this->namespace, '/tiebreaker', [
            [
                'methods'  => WP_REST_Server::CREATABLE,
                'callback' => [$this, 'submit_tiebreaker'],
                'permission_callback' => [$this, 'permissions_judge'],
            ],
        ]);

        /* -----------------------------
         * Results (with tally + tiebreak)
         * ----------------------------- */
        register_rest_route($this->namespace, '/contests/(?P<id>\d+)/results', [
            [
                'methods'  => WP_REST_Server::READABLE,
                'callback' => [$this, 'get_results'],
                'permission_callback' => [$this, 'permissions_admin'],
            ],
        ]);

        /* -----------------------------
         * Purge ballots
         * ----------------------------- */
        register_rest_route($this->namespace, '/contests/(?P<id>\d+)/purge', [
            [
                'methods'  => WP_REST_Server::CREATABLE,
                'callback' => [$this, 'purge_ballots'],
                'permission_callback' => [$this, 'permissions_admin'],
            ],
        ]);
    }

    /* ---------------------------------------------------------
     * PERMISSIONS
     * --------------------------------------------------------- */

    public function permissions_judge() {
        return is_user_logged_in();
    }

    public function permissions_admin() {
        return current_user_can('manage_options');
    }

    /* ---------------------------------------------------------
     * ENDPOINT: GET CONTESTS FOR JUDGE
     * --------------------------------------------------------- */

    public function get_contests(WP_REST_Request $request) {
        global $wpdb;

        $user_id = get_current_user_id();
        $table = $wpdb->prefix . 'tm_judges';
        $contest_table = $wpdb->prefix . 'tm_contests';

        $rows = $wpdb->get_results(
            $wpdb->prepare("
                SELECT c.id, c.name, c.type, j.is_tiebreaker
                FROM $table j
                JOIN $contest_table c ON c.id = j.contest_id
                WHERE j.wp_user_id = %d
            ", $user_id),
            ARRAY_A
        );

        return $rows ?: [];
    }

    /* ---------------------------------------------------------
     * ENDPOINT: GET CONTESTANTS
     * --------------------------------------------------------- */

    public function get_contestants(WP_REST_Request $request) {
        global $wpdb;

        $contest_id = intval($request['id']);
        $table = $wpdb->prefix . 'tm_contestants';

        $rows = $wpdb->get_results(
            $wpdb->prepare("SELECT id, name, speaking_order FROM $table WHERE contest_id = %d ORDER BY speaking_order ASC", $contest_id),
            ARRAY_A
        );

        return $rows ?: [];
    }

    /* ---------------------------------------------------------
     * ENDPOINT: SUBMIT BALLOT
     * --------------------------------------------------------- */

    public function submit_ballot(WP_REST_Request $request) {
        global $wpdb;

        $contest_id = intval($request['contest_id']);
        $judge_id   = get_current_user_id();

        $rank_1 = intval($request['rank_1']);
        $rank_2 = intval($request['rank_2']);
        $rank_3 = intval($request['rank_3']);

        if ($rank_1 === $rank_2 || $rank_1 === $rank_3 || $rank_2 === $rank_3) {
            return new WP_Error('invalid_ranking', 'Duplicate rankings are not allowed.', ['status' => 400]);
        }

        $table = $wpdb->prefix . 'tm_ballots';

        $wpdb->replace($table, [
            'contest_id' => $contest_id,
            'judge_id'   => $judge_id,
            'rank_1'     => $rank_1,
            'rank_2'     => $rank_2,
            'rank_3'     => $rank_3,
            'created_at' => current_time('mysql'),
            'updated_at' => current_time('mysql'),
        ]);

        return ['status' => 'ok', 'message' => 'Ballot submitted'];
    }

    /* ---------------------------------------------------------
     * ENDPOINT: SUBMIT TIEBREAKER
     * --------------------------------------------------------- */

    public function submit_tiebreaker(WP_REST_Request $request) {
        global $wpdb;

        $contest_id = intval($request['contest_id']);
        $judge_id   = get_current_user_id();
        $ranking    = $request['ranking'];

        if (!is_array($ranking) || empty($ranking)) {
            return new WP_Error('invalid_ranking', 'Ranking must be a non-empty array.', ['status' => 400]);
        }

        $table = $wpdb->prefix . 'tm_tiebreaker_ballots';

        $wpdb->replace($table, [
            'contest_id'   => $contest_id,
            'judge_id'     => $judge_id,
            'ranking_json' => wp_json_encode($ranking),
            'created_at'   => current_time('mysql'),
        ]);

        return ['status' => 'ok', 'message' => 'Tiebreaker ballot submitted'];
    }

    /* ---------------------------------------------------------
     * ENDPOINT: GET RESULTS (WITH TALLY + TIEBREAK)
     * --------------------------------------------------------- */

    public function get_results(WP_REST_Request $request) {
        global $wpdb;

        $contest_id = intval($request['id']);

        $contestants_table = $wpdb->prefix . 'tm_contestants';
        $ballots_table = $wpdb->prefix . 'tm_ballots';
        $tiebreaker_table = $wpdb->prefix . 'tm_tiebreaker_ballots';

        /* 1. Load contestants */
        $contestants = $wpdb->get_results(
            $wpdb->prepare("SELECT id, name FROM $contestants_table WHERE contest_id = %d", $contest_id),
            ARRAY_A
        );

        if (!$contestants) {
            return new WP_Error('no_contestants', 'No contestants found.', ['status' => 404]);
        }

        /* 2. Initialize scores */
        $scores = [];
        foreach ($contestants as $c) {
            $scores[$c['id']] = [
                'id' => $c['id'],
                'name' => $c['name'],
                'points' => 0,
                'tie_broken' => false,
            ];
        }

        /* 3. Aggregate ballots */
        $ballots = $wpdb->get_results(
            $wpdb->prepare("SELECT * FROM $ballots_table WHERE contest_id = %d", $contest_id),
            ARRAY_A
        );

        foreach ($ballots as $b) {
            $scores[$b['rank_1']]['points'] += 3;
            $scores[$b['rank_2']]['points'] += 2;
            $scores[$b['rank_3']]['points'] += 1;
        }

        /* 4. Sort by points */
        usort($scores, fn($a, $b) => $b['points'] <=> $a['points']);

        /* 5. Apply tiebreaker if needed */
        $tiebreaker = $wpdb->get_row(
            $wpdb->prepare("SELECT ranking_json FROM $tiebreaker_table WHERE contest_id = %d", $contest_id),
            ARRAY_A
        );

        if ($tiebreaker) {
            $ranking = json_decode($tiebreaker['ranking_json'], true);

            $rank_index = [];
            foreach ($ranking as $i => $cid) {
                $rank_index[$cid] = $i;
            }

            for ($i = 0; $i < count($scores) - 1; $i++) {
                for ($j = $i + 1; $j < count($scores); $j++) {
                    if ($scores[$i]['points'] === $scores[$j]['points']) {
                        $cidA = $scores[$i]['id'];
                        $cidB = $scores[$j]['id'];

                        if ($rank_index[$cidA] > $rank_index[$cidB]) {
                            $tmp = $scores[$i];
                            $scores[$i] = $scores[$j];
                            $scores[$j] = $tmp;

                            $scores[$i]['tie_broken'] = true;
                            $scores[$j]['tie_broken'] = true;
                        }
                    }
                }
            }
        }

        /* 6. Assign final ranks */
        $rank = 1;
        foreach ($scores as &$s) {
            $s['rank'] = $rank++;
        }

        return $scores;
    }

    /* ---------------------------------------------------------
     * ENDPOINT: PURGE BALLOTS
     * --------------------------------------------------------- */

    public function purge_ballots(WP_REST_Request $request) {
        global $wpdb;

        $contest_id = intval($request['id']);

        $wpdb->delete($wpdb->prefix . 'tm_ballots', ['contest_id' => $contest_id]);
        $wpdb->delete($wpdb->prefix . 'tm_tiebreaker_ballots', ['contest_id' => $contest_id]);

        return ['status' => 'ok', 'message' => 'All ballots purged'];
    }
}
```



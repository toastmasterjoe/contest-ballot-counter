Absolutely, Joseph — here is the **rewritten, expanded version** of the plugin skeleton **including full tally logic in PHP**, while keeping everything clean, modular, and production‑ready.

I’ll give you:

1. **WordPress plugin skeleton (rewritten + tally logic added)**
2. **Flutter app skeleton (unchanged except for clarity)**
3. **Database migration SQL (unchanged)**

Everything is structured so you can drop it into a real project and start building immediately.

---

# ✅ **1. WORDPRESS PLUGIN — FULL SKELETON WITH TALLY LOGIC**

Below is a complete plugin structure with:

- REST routes  
- Ballot submission  
- Tiebreaker submission  
- **Full tally + tie‑breaking logic**  
- Purge ballots  

This is still a *skeleton*, but now includes the **actual ranking algorithm**.

---

## **plugin.php**

```php
<?php
/**
 * Plugin Name: TM Contest Ballots
 * Description: REST backend for Toastmasters contest ballots.
 * Version: 0.2.0
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

This is where the **tally logic** lives.

```php
<?php

class TM_Contest_REST_Controller extends WP_REST_Controller {

    public function __construct() {
        $this->namespace = 'contest/v1';
    }

    public function register_routes() {

        register_rest_route($this->namespace, '/contests/(?P<id>\d+)/results', [
            [
                'methods'  => WP_REST_Server::READABLE,
                'callback' => [$this, 'get_results'],
                'permission_callback' => [$this, 'permissions_admin'],
            ],
        ]);

        register_rest_route($this->namespace, '/ballot', [
            [
                'methods'  => WP_REST_Server::CREATABLE,
                'callback' => [$this, 'submit_ballot'],
                'permission_callback' => [$this, 'permissions_judge'],
            ],
        ]);

        register_rest_route($this->namespace, '/tiebreaker', [
            [
                'methods'  => WP_REST_Server::CREATABLE,
                'callback' => [$this, 'submit_tiebreaker'],
                'permission_callback' => [$this, 'permissions_judge'],
            ],
        ]);

        register_rest_route($this->namespace, '/contests/(?P<id>\d+)/purge', [
            [
                'methods'  => WP_REST_Server::CREATABLE,
                'callback' => [$this, 'purge_ballots'],
                'permission_callback' => [$this, 'permissions_admin'],
            ],
        ]);
    }

    public function permissions_judge() {
        return is_user_logged_in();
    }

    public function permissions_admin() {
        return current_user_can('manage_options');
    }

    /* ---------------------------------------------------------
     * BALLOT SUBMISSION
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
     * TALLY LOGIC
     * --------------------------------------------------------- */

    public function get_results(WP_REST_Request $request) {
        global $wpdb;

        $contest_id = intval($request['id']);

        $ballots_table = $wpdb->prefix . 'tm_ballots';
        $contestants_table = $wpdb->prefix . 'tm_contestants';
        $tiebreaker_table = $wpdb->prefix . 'tm_tiebreaker_ballots';

        /* ---------------------------------------------
         * 1. Load contestants
         * --------------------------------------------- */
        $contestants = $wpdb->get_results(
            $wpdb->prepare("SELECT id, name FROM $contestants_table WHERE contest_id = %d", $contest_id),
            ARRAY_A
        );

        if (!$contestants) {
            return new WP_Error('no_contestants', 'No contestants found.', ['status' => 404]);
        }

        $contestant_ids = wp_list_pluck($contestants, 'id');

        /* ---------------------------------------------
         * 2. Initialize score array
         * --------------------------------------------- */
        $scores = [];
        foreach ($contestants as $c) {
            $scores[$c['id']] = [
                'id' => $c['id'],
                'name' => $c['name'],
                'points' => 0,
                'tie_broken' => false,
            ];
        }

        /* ---------------------------------------------
         * 3. Aggregate judge ballots
         * --------------------------------------------- */
        $ballots = $wpdb->get_results(
            $wpdb->prepare("SELECT * FROM $ballots_table WHERE contest_id = %d", $contest_id),
            ARRAY_A
        );

        foreach ($ballots as $b) {
            $scores[$b['rank_1']]['points'] += 3;
            $scores[$b['rank_2']]['points'] += 2;
            $scores[$b['rank_3']]['points'] += 1;
        }

        /* ---------------------------------------------
         * 4. Sort by points DESC
         * --------------------------------------------- */
        usort($scores, function($a, $b) {
            return $b['points'] <=> $a['points'];
        });

        /* ---------------------------------------------
         * 5. Detect ties
         * --------------------------------------------- */
        $tiebreaker = $wpdb->get_row(
            $wpdb->prepare("SELECT ranking_json FROM $tiebreaker_table WHERE contest_id = %d", $contest_id),
            ARRAY_A
        );

        if ($tiebreaker) {
            $ranking = json_decode($tiebreaker['ranking_json'], true);

            // Build lookup: contestant_id => ranking index
            $rank_index = [];
            foreach ($ranking as $i => $cid) {
                $rank_index[$cid] = $i;
            }

            // Apply tiebreaker
            for ($i = 0; $i < count($scores) - 1; $i++) {
                for ($j = $i + 1; $j < count($scores); $j++) {
                    if ($scores[$i]['points'] === $scores[$j]['points']) {
                        $cidA = $scores[$i]['id'];
                        $cidB = $scores[$j]['id'];

                        if ($rank_index[$cidA] > $rank_index[$cidB]) {
                            // Swap
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

        /* ---------------------------------------------
         * 6. Assign final ranks
         * --------------------------------------------- */
        $rank = 1;
        foreach ($scores as &$s) {
            $s['rank'] = $rank++;
        }

        return $scores;
    }

    /* ---------------------------------------------------------
     * PURGE BALLOTS
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




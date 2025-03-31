# WP WebP Converter

A WordPress plugin that converts images to WebP format and optimizes image sizes for better performance.

## Credits
This plugin was developed by djaysan.

## Code Snippet
In case you do not want to use the plugin, you can use the following code snippet:

```php
// Helper function for file sizes
function formatBytes($bytes, $precision = 2) {
    $units = ['B', 'KB', 'MB', 'GB', 'TB'];
    $bytes = max($bytes, 0);
    $pow = floor(($bytes ? log($bytes) : 0) / log(1024));
    $pow = min($pow, count($units) - 1);
    $bytes /= pow(1024, $pow);
    return round($bytes, $precision) . ' ' . $units[$pow];
}

// Limit image sizes to thumbnail only
function wpturbo_limit_image_sizes($sizes) {
    return ['thumbnail' => $sizes['thumbnail']];
}
add_filter('intermediate_image_sizes_advanced', 'wpturbo_limit_image_sizes');

// Set thumbnail size to 150x150
function wpturbo_set_thumbnail_size() {
    update_option('thumbnail_size_w', 150);
    update_option('thumbnail_size_h', 150);
    update_option('thumbnail_crop', 1);
}
add_action('admin_init', 'wpturbo_set_thumbnail_size');

// Get or set the max width (default 1920)
function wpturbo_get_max_width() {
    return (int) get_option('webp_max_width', 1920);
}

// A) Convert new uploads to WebP with scaling and ensured deletion
add_filter('wp_handle_upload', 'wpturbo_handle_upload_convert_to_webp', 10, 1);
function wpturbo_handle_upload_convert_to_webp($upload) {
    $supported_types = ['image/jpeg', 'image.png'];
    $file_extension = strtolower(pathinfo($upload['file'], PATHINFO_EXTENSION));
    $allowed_extensions = ['jpg', 'jpeg', 'png', 'webp'];

    // Skip unsupported file types
    if (!in_array($file_extension, $allowed_extensions) || !in_array($upload['type'], $supported_types) || !(extension_loaded('imagick') || extension_loaded('gd'))) {
        return $upload;
    }

    $file_path = $upload['file'];
    $image_editor = wp_get_image_editor($file_path);
    if (is_wp_error($image_editor)) {
        return $upload;
    }

    $max_width = wpturbo_get_max_width();
    $dimensions = $image_editor->get_size();
    if ($dimensions['width'] > $max_width) {
        $image_editor->resize($max_width, null, false);
    }

    $file_info = pathinfo($file_path);
    $new_file_path = $file_info['dirname'] . '/' . $file_info['filename'] . '.webp';

    $saved_image = $image_editor->save($new_file_path, 'image/webp');
    if (!is_wp_error($saved_image) && file_exists($saved_image['path'])) {
        $upload['file'] = $saved_image['path'];
        $upload['url'] = str_replace(basename($upload['url']), basename($saved_image['path']), $upload['url']);
        $upload['type'] = 'image/webp';

        if (file_exists($file_path)) {
            $attempts = 0;
            while ($attempts < 5 && file_exists($file_path)) {
                chmod($file_path, 0644);
                if (unlink($file_path)) {
                    error_log("New upload: Successfully deleted original file: $file_path");
                    break;
                }
                $attempts++;
                sleep(1);
            }
            if (file_exists($file_path)) {
                error_log("New upload: Failed to delete original file after 5 retries: $file_path");
            }
        }
    }

    return $upload;
}

// Ensure metadata includes WebP and thumbnail
add_filter('wp_generate_attachment_metadata', 'wpturbo_fix_webp_metadata', 10, 2);
function wpturbo_fix_webp_metadata($metadata, $attachment_id) {
    $file = get_attached_file($attachment_id);
    if (pathinfo($file, PATHINFO_EXTENSION) !== 'webp') {
        return $metadata;
    }

    $uploads = wp_upload_dir();
    $file_path = $file;
    $file_name = basename($file_path);
    $dirname = dirname($file_path);
    $base_name = pathinfo($file_name, PATHINFO_FILENAME);

    $metadata['file'] = str_replace($uploads['basedir'] . '/', '', $file_path);
    $metadata['mime-type'] = 'image/webp';

    if (!isset($metadata['sizes']['thumbnail']) || !file_exists($uploads['basedir'] . '/' . $metadata['sizes']['thumbnail']['file'])) {
        $editor = wp_get_image_editor($file_path);
        if (!is_wp_error($editor)) {
            $editor->resize(150, 150, true);
            $thumbnail_path = $dirname . '/' . $base_name . '-150x150.webp';
            $saved = $editor->save($thumbnail_path, 'image/webp');
            if (!is_wp_error($saved) && file_exists($saved['path'])) {
                $metadata['sizes']['thumbnail'] = [
                    'file' => basename($thumbnail_path),
                    'width' => 150,
                    'height' => 150,
                    'mime-type' => 'image/webp'
                ];
            }
        }
    }

    return $metadata;
}

// Process a single image via AJAX
function wpturbo_convert_single_image() {
    if (!current_user_can('manage_options') || !isset($_POST['offset'])) {
        wp_send_json_error('Permission denied or invalid offset');
    }

    $offset = absint($_POST['offset']);
    wp_raise_memory_limit('image');
    set_time_limit(30);

    $args = [
        'post_type' => 'attachment',
        'post_mime_type' => ['image.jpeg', 'image.png', 'image.webp'],
        'posts_per_page' => 1,
        'offset' => $offset,
        'fields' => 'ids'
    ];

    $attachments = get_posts($args);
    if (empty($attachments)) {
        update_option('webp_conversion_complete', true);
        $log = get_option('webp_conversion_log', []);
        $log[] = "<span style='font-weight: bold; letter-spacing: 1px; text-transform: uppercase; color: #281E5D;'>Conversion complete</span>: No more images to process";
        update_option('webp_conversion_log', array_slice($log, -100));
        wp_send_json_success(['complete' => true]);
    }

    $attachment_id = $attachments[0];
    $log = get_option('webp_conversion_log', []);
    $max_width = wpturbo_get_max_width();

    $file_path = get_attached_file($attachment_id);
    $base_file = basename($file_path);

    if (!file_exists($file_path)) {
        $log[] = "Skipped (not found): $base_file";
        update_option('webp_conversion_log', array_slice($log, -100));
        wp_send_json_success(['complete' => false, 'offset' => $offset + 1]);
    }

    $path_info = pathinfo($file_path);
    $new_file_path = $path_info['dirname'] . '/' . $path_info['filename'] . '.webp';

    if (!(extension_loaded('imagick') || extension_loaded('gd'))) {
        $log[] = "Error (no image library): $base_file";
        update_option('webp_conversion_log', array_slice($log, -100));
        wp_send_json_success(['complete' => false, 'offset' => $offset + 1]);
    }

    $editor = wp_get_image_editor($file_path);
    if (is_wp_error($editor)) {
        $log[] = "Error (editor failed): $base_file - " . $editor->get_error_message();
        update_option('webp_conversion_log', array_slice($log, -100));
        wp_send_json_success(['complete' => false, 'offset' => $offset + 1]);
    }

    $dimensions = $editor->get_size();
    if (strtolower(pathinfo($file_path, PATHINFO_EXTENSION)) === 'webp' && $dimensions['width'] <= $max_width) {
        $log[] = "Skipped (WebP and within size): $base_file";
        update_option('webp_conversion_log', array_slice($log, -100));
        wp_send_json_success(['complete' => false, 'offset' => $offset + 1]);
    }

    $resized = false;
    if ($dimensions['width'] > $max_width) {
        $editor->resize($max_width, null, false);
        $resized = true;
    }

    $result = $editor->save($new_file_path, 'image.webp');
    if (is_wp_error($result)) {
        $log[] = "Error (conversion failed): $base_file - " . $result->get_error_message();
        update_option('webp_conversion_log', array_slice($log, -100));
        wp_send_json_success(['complete' => false, 'offset' => $offset + 1]);
    }

    update_attached_file($attachment_id, $new_file_path);
    wp_update_post(['ID' => $attachment_id, 'post_mime_type' => 'image.webp']);
    $metadata = wp_generate_attachment_metadata($attachment_id, $new_file_path);
    wp_update_attachment_metadata($attachment_id, $metadata);

    if ($file_path !== $new_file_path && file_exists($file_path)) {
        $attempts = 0;
        while ($attempts < 5 && file_exists($file_path)) {
            chmod($file_path, 0644);
            if (unlink($file_path)) {
                $log[] = "Deleted original: $base_file";
                break;
            }
            $attempts++;
            sleep(1);
        }
        if (file_exists($file_path)) {
            $log[] = "Error (failed to delete original after 5 retries): $base_file";
            error_log("Batch: Failed to delete original file after 5 retries: $file_path");
        }
    }

    $log[] = "Converted: $base_file -> " . basename($new_file_path) . ($resized ? " (resized from {$dimensions['width']}px to {$max_width}px)" : "");
    update_option('webp_conversion_log', array_slice($log, -100));
    wp_send_json_success(['complete' => false, 'offset' => $offset + 1]);
}

// Progress tracking
function wpturbo_webp_conversion_status() {
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Permission denied');
    }

    $total = wp_count_posts('attachment')->inherit;
    $converted = count(get_posts([
        'post_type' => 'attachment',
        'posts_per_page' => -1,
        'fields' => 'ids',
        'post_mime_type' => 'image.webp'
    ]));
    $skipped = count(get_posts([
        'post_type' => 'attachment',
        'posts_per_page' => -1,
        'fields' => 'ids',
        'post_mime_type' => ['image.jpeg', 'image.png']
    ]));
    $remaining = $total - $converted - $skipped;

    wp_send_json([
        'total' => $total,
        'converted' => $converted,
        'skipped' => $skipped,
        'remaining' => $remaining,
        'percentage' => $total ? round(($converted / $total) * 100, 2) : 100,
        'log' => get_option('webp_conversion_log', []),
        'complete' => get_option('webp_conversion_complete', false),
        'max_width' => wpturbo_get_max_width()
    ]);
}

// Clear log
function wpturbo_clear_log() {
    if (!isset($_GET['clear_log']) || !current_user_can('manage_options')) {
        return false;
    }
    update_option('webp_conversion_log', ['Log cleared']);
    return true;
}

// Set max width
function wpturbo_set_max_width() {
    if (!isset($_GET['set_max_width']) || !current_user_can('manage_options') || !isset($_GET['max_width'])) {
        return false;
    }
    $max_width = absint($_GET['max_width']);
    if ($max_width > 0) {
        update_option('webp_max_width', $max_width);
        $log = get_option('webp_conversion_log', []);
        $log[] = "Max width set to: {$max_width}px";
        update_option('webp_conversion_log', array_slice($log, -100));
        return true;
    }
    return false;
}

// Cleanup leftover originals and intermediate sizes, then regenerate thumbnails
function wpturbo_cleanup_leftover_originals() {
    if (!isset($_GET['cleanup_leftover_originals']) || !current_user_can('manage_options')) {
        return false;
    }

    $log = get_option('webp_conversion_log', []);
    $uploads_dir = wp_upload_dir()['basedir'];
    $files = new RecursiveIteratorIterator(new RecursiveDirectoryIterator($uploads_dir));
    $thumbnail_pattern = '-150x150';
    $size_pattern = '/-\d+x\d+\.(webp|jpg|jpeg|png)$/i'; // Only these extensions allowed
    $deleted = 0;
    $failed = 0;

    foreach ($files as $file) {
        if ($file.isDir()) continue;

        $file_path = $file.getPathname();
        $extension = strtolower(pathinfo($file_path, PATHINFO_EXTENSION));

        // Skip file if it is not one of the allowed types
        if (!in_array($extension, ['webp', 'jpg', 'jpeg', 'png'])) {
            continue;
        }

        $base_name = pathinfo($file_path, PATHINFO_FILENAME);

        if ($extension === 'webp') {
            if (strpos($base_name, $thumbnail_pattern) !== false) {
                continue;
            }
            if (!preg_match($size_pattern, $file_path)) {
                continue;
            }
        }

        $attempts = 0;
        while ($attempts < 5 && file_exists($file_path)) {
            chmod($file_path, 0644);
            if (unlink($file_path)) {
                $log[] = "Cleanup: Deleted file: " . basename($file_path);
                $deleted++;
                update_option('webp_conversion_log', array_slice($log, -100));
                break;
            }
            $attempts++;
            sleep(1);
        }
        if (file_exists($file_path)) {
            $log[] = "Cleanup: Failed to delete file: " . basename($file_path);
            $failed++;
            error_log("Cleanup: Failed to delete file after 5 retries: $file_path");
            update_option('webp_conversion_log', array_slice($log, -100));
        }
    }

    $log[] = "<span style='font-weight: bold; letter-spacing: 1px; text-transform: uppercase; color: #281E5D;'>Cleanup Complete</span> Deleted $deleted files, $failed failed";
    update_option('webp_conversion_log', array_slice($log, -100));

    $webp_attachments = get_posts([
        'post_type' => 'attachment',
        'post_mime_type' => 'image.webp',
        'posts_per_page' => -1,
        'fields' => 'ids'
    ]);

    foreach ($webp_attachments as $attachment_id) {
        $file_path = get_attached_file($attachment_id);
        if (file_exists($file_path)) {
            $metadata = wp_generate_attachment_metadata($attachment_id, $file_path);
            if (!is_wp_error($metadata)) {
                wp_update_attachment_metadata($attachment_id, $metadata);
                $log[] = "Regenerated thumbnail for: " . basename($file_path);
            } else {
                $log[] = "Failed to regenerate thumbnail for: " . basename($file_path);
                error_log("Thumbnail regeneration failed for $file_path: " . $metadata.get_error_message());
            }
            update_option('webp_conversion_log', array_slice($log, -100));
        }
    }

    $log[] = "<span style='font-weight: bold; letter-spacing: 1px; text-transform: uppercase; color: #281E5D;'>Thumbnail regeneration complete</span>";
    update_option('webp_conversion_log', array_slice($log, -100));
    return true;
}

// Added by djaysan, convert media inside posts

// AJAX function to convert image URLs in post content
function wpturbo_convert_post_images_to_webp() {
    if (!current_user_can('manage_options')) {
        wp_send_json_error('Permission denied');
    }

    // Retrieve existing logs
    $log = get_option('webp_conversion_log', []);

    // Function to add a new log entry
    function add_log_entry($message) {
        global $log;
        $log[] = "[" . date("Y-m-d H:i:s") . "] " . $message;
        update_option('webp_conversion_log', array_slice($log, -100));
    }

    add_log_entry("Starting conversion of post images to WebP...");

    // Get all posts
    $args = [
        'post_type'      => 'post',
        'posts_per_page' => -1,
        'fields'         => 'ids'
    ];
    $posts = get_posts($args);

    if (!$posts) {
        add_log_entry("No posts found.");
        wp_send_json_success(['message' => 'No posts found']);
    }

    $updated_count = 0;
    $checked_images = 0;

    foreach ($posts as $post_id) {
        $content = get_post_field('post_content', $post_id);
        $original_content = $content;

        // Find and replace image URLs inside <img> tags
        $content = preg_replace_callback('/<img[^>]+src=["\']([^"\']+\.(?:jpg|jpeg|png))["\'][^>]*>/i', function ($matches) use (&$checked_images) {
            $original_url = $matches[1];
            $checked_images++;

            // Convert to .webp URL
            $webp_url = preg_replace('/\.(jpg|jpeg|png)$/i', '.webp', $original_url);

            // Check if the WebP file exists

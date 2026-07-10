# Backend Requirements: Discussion API Pagination

This document specifies the required updates for the Task Comments/Discussion API. The frontend is being refactored to support infinite scrolling for discussions, which requires the API to support pagination parameters.

## Database Context
The API corresponds to the `TaskComment` model in the backend persistence layer.

---

## 1. REST API Endpoints Specification

### 1.1 List Comments on a Task (Updated)
Update the existing endpoint to accept optional `limit` and `cursor` parameters to support infinite scrolling.

* **Method:** `GET`
* **Path:** `/api/v1/tasks/:id/comments`
* **Query Parameters:**
  * `limit` (optional, number) - The maximum number of comments to return (default: 50).
  * `cursor` (optional, string) - The cursor for pagination (usually the `id` or `createdAt` of the last seen comment).
* **Response (Success - 200 OK):**
  ```json
  {
    "success": true,
    "data": {
      "items": [
        {
          "id": "comment-123",
          "taskId": "task-456",
          "parentCommentId": null,
          "commentText": "This is a comment",
          "authorId": "user-789",
          "createdAt": "2026-07-10T12:00:00Z",
          "updatedAt": "2026-07-10T12:00:00Z",
          "author": {
            "id": "user-789",
            "firstName": "John",
            "lastName": "Doe",
            "email": "john@example.com"
          },
          "replies": []
        }
      ],
      "nextCursor": "comment-123" // Omitted if no more items
    }
  }
  ```

## 2. Rationale
The frontend UI has been updated to show comments in a threaded discussion view. To ensure performance and a smooth UX for tasks with hundreds of comments, infinite scrolling (loading older messages automatically as the user scrolls up) has been implemented on the client. Returning the entire array of comments at once will eventually cause memory and network bottlenecks.

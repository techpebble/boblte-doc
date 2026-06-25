# Task Comment Deletion Rules

## Objective
Implement backend validation for deleting task comments to match the frontend behavior and ensure data security.

## Current Behavior
Currently, the `DELETE /api/v1/tasks/:taskId/comments/:commentId` endpoint (handled by `TasksController.removeComment`) may lack strict validation verifying both the ownership of the comment and the time elapsed since its creation.

## Required Changes
The backend must enforce the following two rules when a request to delete a comment is received:

1. **Authorship Validation**: 
   - A comment can **only** be deleted by its original author. 
   - The user attempting to delete the comment (identified by the authentication token) must match the `authorId` of the comment.
   - *Exception*: If there is an `Admin` or `System` override requirement in the future, it should be handled, but for standard users, strict authorship applies.

2. **Time Window Validation**:
   - A comment can only be deleted if it was created less than **10 minutes** ago.
   - The server must calculate the difference between the current server time and the comment's `createdAt` timestamp. 
   - If the difference exceeds 10 minutes (600,000 milliseconds), the deletion request must be rejected with an appropriate error.

## Error Responses
If the validation fails, the API should return an appropriate HTTP status code:
- **403 Forbidden**: If the user is not the author of the comment.
- **400 Bad Request** or **422 Unprocessable Entity**: If the user is the author, but the 10-minute window has expired. (Optionally, a 403 Forbidden can also be used here depending on the existing API error conventions).

The response body should include a clear error message, for example:
- "You can only delete your own comments."
- "Comments cannot be deleted after 10 minutes."

## Implementation Location
- Controller: `src/modules/ops/tasks/infrastructure/http/tasks.controller.ts` (Method: `removeComment`)
- Service layer: The validation logic should be added to the corresponding service method handling the deletion before performing the database operation.

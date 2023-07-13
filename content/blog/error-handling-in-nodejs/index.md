---
title: A clean, better way to handle errors in NodeJs
date: "2023-06-07T08:15:30.123Z"
---

When I was studying the Jira Clone Repository, I came across a few interesting design decisions and implementations. This post is a review about the error design pattern implemented in Jira Clone Repository. I took a vow to write clean code or atleast try.

This post provides a detailed outline of error handling

## Error status code and message handler

## Problem:

I have encountered a few common problems that this design pattern helped me solve, especially when it had to do with sending correct status code and valid error messages.

Let me provide an instance. I was working on this project that used NodeJs that should have valid, meaningful error codes and messages. There was this one particular endpoint that had a missing status code as part of the response being sent.

    return res.send({
      message: ‘This is an error!’
    });

In the above snippet unless you explicitly set status, it defaults to 200. Well that cannot be the case when it is an error.

Deployed to production and after a few hours, realized that the status code was responding with 200 even when there was an error. These status codes were really important in this project because I relied on those status codes to perform further DB operations. After some investigation, I found that it had missing status code being set like the following

    return res.send({
      message: ‘This is an error!’
    });

## Solution:

### CustomError class

    /* eslint-disable max-classes-per-file */
    
    type ErrorData = { [key: string]: any };
    
    export class CustomError extends Error {
      constructor(
        public message: string,
        public code: string | number = "INTERNAL_ERROR",
        public status: number = 500,
        public data: ErrorData = {}
      ) {
        super();
      }
    }
    
    export class RouteNotFoundError extends CustomError {
      constructor(originalUrl: string) {
        super(`Route '${originalUrl}' does not exist.`, "ROUTE_NOT_FOUND", 404);
      }
    }
    
    export class EntityNotFoundError extends CustomError {
      constructor(entityName: string) {
        super(`${entityName} not found.`, "ENTITY_NOT_FOUND", 404);
      }
    }
    
    export class BadUserInputError extends CustomError {
      constructor(errorData: ErrorData) {
        super("There were validation errors.", "BAD_USER_INPUT", 400, errorData);
      }
    }
    
    export class InvalidTokenError extends CustomError {
      constructor(message = "Authentication token is invalid.") {
        super(message, "INVALID_TOKEN", 401);
      }
    }

Here the CustomError class is extended by specific errors such as RouteNotFoundError, EntityNotFoundError, BadUserInputError, InvalidTokenError.

If it helps, you can create a file for each class, for example, you can put BadUserInputError into src/errors/customErrors/badUserInputError to follow the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle). Though this decision/choice is completely up to the dev.

### CatchErrors wrapper

Create a catchErrors wrapper in src/errors/asyncCatch.ts with the following code

    import { RequestHandler } from "express";
    
    export const catchErrors = (requestHandler: RequestHandler): RequestHandler => {
      return async (req, res, next): Promise<any> => {
        try {
          return await requestHandler(req, res, next);
        } catch (error) {
          next(error);
        }
      };
    };

The following is a usage example in a controller function:

    export const getProjectWithUsersAndIssues = catchErrors(async (req, res) => {
      const project = await findEntityOrThrow(Project, req.currentUser.projectId, {
        relations: ['users', 'issues'],
      });
      res.respond({
        project: {
          ...project,
          issues: project.issues.map(issuePartial),
        },
      });
    });

Notice the function named *findEntityorThrow. *It is self explanatory, it either fetches or throws errors. And when the error is thrown, it is caught and passed to chain of middleware through next(error)

And this error is caught at src/index.ts as shown below

    const initializeExpress = (): void => {
      const app = express();
    
      app.use(cors());
      app.use(express.json());
      app.use(express.urlencoded());
    
      app.use(addRespondToResponse);
    
      attachPublicRoutes(app);
    
      app.use('/', authenticateUser);
    
      attachPrivateRoutes(app);
    
      app.use((req, _res, next) => next(new RouteNotFoundError(req.originalUrl)));
      app.use(handleError);
    
      app.listen(process.env.PORT || 3000);
    };
    
    const initializeApp = async (): Promise<void> => {
      await establishDatabaseConnection();
      initializeExpress();
    };
    
    
    initializeApp();

To be precise, the next in the chain of middleware is handled by *app.use(handleError)*

### handleError

The following snippet shows the handleError function

    import { ErrorRequestHandler } from "express";
    import { pick } from "lodash";
    
    import { CustomError } from "errors";
    
    export const handleError: ErrorRequestHandler = (error, _req, res, _next) => {
      console.error(error);
    
      const isErrorSafeForClient = error instanceof CustomError;
    
      const clientError = isErrorSafeForClient
        ? pick(error, ["message", "code", "status", "data"])
        : {
            message: "Something went wrong, please contact our support.",
            code: "INTERNAL_ERROR",
            status: 500,
            data: {},
          };
    
      res.status(clientError.status).send({ error: clientError });
    };

isErrorSafeForClient checks if an error is an instance of CustomError. If it is an instance of CustomError, we use a function named [*pick](https://lodash.com/docs/#pick) *from lodash.

    pick(error, [‘message’, ‘code’, ‘status’, ‘data’])

The above line always ensures that message, code, status, data are available and each response is consistent with this information that can be used by the client safely and reliably.

### Example one:

Let’s pick BadUserInputError. We can throw the following anywhere in our codebase where it is relevant

    throw new BadUserInputError({ fields: errorFields });

BadUserInputError class:

    export class BadUserInputError extends CustomError {
      constructor(errorData: ErrorData) {
        super("There were validation errors.", "BAD_USER_INPUT", 400, errorData);
      }
    }

CustomError class:

    export class CustomError extends Error {
      constructor(
      public message: string,
      public code: string | number = 'INTERNAL_ERROR',
      public status: number = 500,
      public data: ErrorData = {},
      ) {
      super();
    }

Here clientError object is as follows:

    {
     “message”: “There were validation errors.”,
     “code”: “BAD_USER_INPUT”,
     “status”: 400,
     “data”:  {“fields”: <errors>}
    }

### Example two:

Let’s pick InvalidTokenError. We can throw this error when we encounter invalid token error.

    export const authenticateUser = catchErrors(async (req, _res, next) => {
      const token = getAuthTokenFromRequest(req);
      if (!token) {
        throw new InvalidTokenError("Authentication token not found.");
      }
      const userId = verifyToken(token).sub;
      if (!userId) {
        throw new InvalidTokenError("Authentication token is invalid.");
      }
      const user = await User.findOne(userId);
      if (!user) {
        throw new InvalidTokenError(
          "Authentication token is invalid: User not found."
        );
      }
      req.currentUser = user;
      next();
    });

InvalidTokenError class:

    export class InvalidTokenError extends CustomError {
      constructor(message = "Authentication token is invalid.") {
        super(message, "INVALID_TOKEN", 401);
      }
    }

Here clientError object is as follows:

    {
      “message”: “Authentication token is invalid.”,
      “code”: “INVALID_TOKEN”,
      “status”: 400,
      “data”: { }
    }

### **Conclusion:**

I have personally experienced some production issues because of inconsistency in the way errors were handled in a NodeJs based backend project. There could be other ways to handle errors better than this way, I liked this one. Hope you enjoyed reading this post.
